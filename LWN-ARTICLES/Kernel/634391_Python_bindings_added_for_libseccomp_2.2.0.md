---
title: Python bindings added for libseccomp 2.2.0
url: https://lwn.net/Articles/634391/
date: "March 4, 2015"
category: "Security-seccomp"
author: "By Jake Edge March 4, 2015"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
March 4, 2015

The secure computing (seccomp) facility was added to Linux some [ten years ago](<https://lwn.net/Articles/120647/>) as a way to restrict programs so that they can only make a small subset of system calls. It is a way to sandbox processes but, over the years, it was found to be _too_ restrictive. Thus, after a few false starts, a new [seccomp mode](<https://lwn.net/Articles/475043/>) that used the kernel's Berkeley Packet Filter (BPF) implementation to provide a way to more flexibly sandbox processes came about in 2012. To help application writers use the facility, Paul Moore [created libseccomp](<https://lwn.net/Articles/494252/>), and he has just [released](<https://lwn.net/Articles/633155/>) version 2.2.0 of the library. 

This is the first release of libseccomp for more than a year, with a 2.1.1 minor release in October 2013 (after 2.1.0 in June 2013). One of the headline features for 2.2.0 is the addition of Python bindings, which have been around for a while but have not been part of a release until now. Other big changes for this release include a switch to [Autotools](<http://en.wikipedia.org/wiki/GNU_build_system>), support for the ARM 64-bit architecture, as well as support for several flavors of the MIPS architecture (mips, mips64, and mips64n32). 

The newer seccomp mode (known as seccomp filters or seccomp mode 2) allows developers to specify which system calls are allowed to be made and to restrict the arguments that can be passed to those system calls. In order to do that, the kernel requires a program written using the [BPF language](<https://lwn.net/Articles/599755/>), which was originally targeted at network filtering, though it has grown well beyond that. Libseccomp is meant to provide a higher-level interface to that functionality—from C and, now, Python. 

If you have a program that is handling untrusted user input—HTTP traffic or image formats, say—you might want to restrict the kinds of operations that program can perform. For example, there might just be a handful of operations that should be allowed to the program, so that if it were compromised by unexpected input, there is little an attacker can actually do. On the other hand, though, the program might require a bit more than the four system calls (`read()`, `write()`, `exit()`, and `sigreturn()`) allowed by the original seccomp mode. 

For instance, `open()` might be allowed, but only to open files for reading. Or, the `write()` system call might be restricted to certain file descriptors (say, 1 and 2 for `stdout` and `stderr`). Meanwhile, all other system calls, including powerful calls that attackers might want to use, such as `execve()` or `socket()`, would be disabled. As with the C interface, the actions taken when a disallowed system call is made depend on how the library is initialized. Those calls could cause the program to be killed, to receive a signal, to generate a `ptrace()` event, or for the call to fail with a particular `errno` value. 

Using the Python bindings is similar in many ways to calling the library directly from C: 

```
import sys, os
        from seccomp import *
    
        f = SyscallFilter(defaction=KILL)
    
        f.add_rule(ALLOW, "open")
        f.add_rule(ALLOW, "write", Arg(0, EQ, sys.stdout.fileno()))
    
        f.load()
    
        x = os.open('/tmp/x', os.O_WRONLY)
        os.write(x, 'Hello, world\n')
```

That, at least conceptually, will create the filter object, add two rules, and load it, which will cause the `write()` to fail. The rules allow the `open()` system call, but only allow calling `write()` on the `stdout` file descriptor. The initialization of the filter object chooses the `KILL` default action, which means the program will be terminated if it uses disallowed system calls. 

However there is a bit more to it than that. When testing the non-error path by commenting out the `os.write()`, Python requires `brk()`, `rt_sigaction()`, and `exit_group()` to exit gracefully. So the following would need to be added to the list of rules: 

```
f.add_rule(ALLOW, "exit_group")
        f.add_rule(ALLOW, "rt_sigaction")
        f.add_rule(ALLOW, "brk")
```

While that does add to the list of allowed system calls, it doesn't really enlarge what an attacker could do when subverting this (extremely simple) program. Using the Python version of `open()` and `write()`, instead of those from the `os` module, requires opening up several more system calls (`mmap()`, `read()`, `close()`, and `fstat()`), which could be a bigger problem. Having both `open()` and `read()` available might allow an attacker to access files, but the contents can only be written to `stdout`, which may well be an impediment. Further refinement of the rules could limit an attacker even further. 

When debugging seccomp filters, there is often a need to track down which system call caused a failure. That can be done a number of different ways. When the `KILL` action is used, as above, the process is forced to exit (with `SIGSYS` as its status), so the shell simply prints "Bad system call". But it also leaves an audit trail that records the system call number that failed: 

```
type=SECCOMP msg=audit(1425421709.486:42015): auid=1000 uid=1000
          gid=1000 ses=1 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
          pid=25113 comm="python" exe="/usr/bin/python2.7" sig=31 arch=c000003e
          **syscall=5** compat=0 ip=0x7faf36e38824 code=0x0
```

One easy way to turn the number reported into a name is by using the `scmp_sys_resolver` tool that is bundled with libseccomp. Another option for determining which system call is causing a failure would be to switch to the `TRAP` action when setting up the filter object, then add a signal handler for the `SIGSYS` signal that gets generated when disallowed system calls are made. Using `ptrace()` or `strace` are possibilities as well. 

Rules can be built using the `Arg()` function to specify the allowable values for up to six arguments. Those values can be tested using the usual comparison operators, but by name (e.g. `EQ` for =, `GE` for >=, `LT` for <, and so on). There is also a `MASKED_EQ` operator that can be used for flag values: 

```
f.add_rule(ALLOW, "open",
                   Arg(1, MASKED_EQ, os.O_RDONLY,
                       os.O_RDONLY | os.O_RDWR | os.O_WRONLY))
```

That rule would ensure that of the three file access flags, only `O_RDONLY` is set, so `open()` would only be allowed for reads. 

The 2.2.0 release comes with a test suite that includes both C and Python versions of each test. It also has man pages for each of the C language `seccomp_*` calls. The only Python language documentation appears to be in the `src/python/seccomp.pyx` file, but perhaps that will change before long. Anyone looking to sandbox their programs should definitely take a peek at libseccomp.
