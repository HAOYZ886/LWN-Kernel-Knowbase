---
title: "Nolibc: a minimal C-library replacement shipped with the kernel"
url: https://lwn.net/Articles/920158/
date: "January 20, 2023"
category: Nolibc
author: "January 20, 2023 This article was contributed by Willy Tarreau"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

January 20, 2023

This article was contributed by Willy Tarreau

The kernel project does not host much user-space code in its repository, but there are exceptions. One of those, currently found in the [`tools/include/nolibc`](<https://elixir.bootlin.com/linux/v6.2-rc4/source/tools/include/nolibc>) directory, has only been present since the 5.1 release. The nolibc project aims to provide minimal C-library emulation for small, low-level workloads. Read on for an overview of nolibc, its history, and future direction written by its principal contributor. 

The nolibc component actually made a discreet entry into the 5.0 kernel as part of the RCU torture-test suite ("rcutorture"), via [commit 66b6f755ad45](<https://git.kernel.org/linus/66b6f755ad45>) ("rcutorture: Import a copy of nolibc"). This happened after Paul McKenney [asked](<https://lwn.net/ml/linux-kernel/20180823174359.GA13033@linux.vnet.ibm.com/>): ""Does anyone do kernel-only deployments, for example, setting up an embedded device having a Linux kernel and absolutely no userspace whatsoever?"" 

He went on: 

> The mkinitramfs approach results in about 40MB of initrd, and dracut about 10MB. Most of this is completely useless for rcutorture, which isn't interested in mounting filesystems, opening devices, and almost all of the other interesting things that mkinitramfs and dracut enable. 
> 
> Those who know me will not be at all surprised to learn that I went overboard making the resulting initrd as small as possible. I started by throwing out everything not absolutely needed by the dash and sleep binaries, which got me down to about 2.5MB, 1.8MB of which was libc. 

This description felt familiar to me, since I have been solving similar problems for a long time. The end result (so far) is nolibc — a minimal C library for times when a system is booted only to run a single tiny program. 

#### A bit of history: when size matters

For 25 years, I have been building single-floppy-based emergency routers, firewalls, and traffic generators containing both a kernel and a root filesystem. I later moved on to credit-card-sized live CDs and, nowadays, embedding tiny interactive shells in all of my kernels to help better recover from boot issues. 

All of these had in common a small program called [preinit](<https://github.com/formilux/flxutils/tree/master/init>) that is in charge of creating `/dev` entries, mounting `/proc`, optionally mounting a RAM-based filesystem, and loading some extra modules before executing `init`. The characteristic that all my kernels have in common is that all of their modules are packaged within the kernel's builtin initial ramfs, reserving the initial RAMdisk (initrd) for the root filesystem. When the kernel boots, it pivots the root and boot mount points to make the pre-packaged modules appear at their final location, making the root filesystem independent of the kernel version used. This is extremely convenient when working from flash or network boot, since you can easily swap kernels without ever touching the filesystem. 

Because it had to fit into 800KB (or smaller) kernels, the preinit code initially had to be minimal; one of the approaches consisted of strictly avoiding stdio or high-level C-library functions and reusing code and data as much as possible (such as merging string tails). This resulted in code that could be statically built and remain small (less than 1KB for the original floppy version) thanks to its small dependencies. 

As it evolved, though, the resulting binary size was often dominated by the C-library initialization code. Switching to [diet libc](<https://www.fefe.de/dietlibc/>) helped but, in 2010, it was not evolving anymore and still had some limitations. I replaced it with the slightly larger — but more active — [uClibc](<https://uclibc.org/>), with a bunch of functions replaced by local alternatives to keep the size low (e.g. 640 bytes were saved on `memmove()` and `strpcy()`), and used this until 2016. 

uClibc, in turn, started to show some more annoying limitations which became a concern when trying to port code to architectures that it did not support by then (such as aarch64), and its maintenance was falling off. I started to consider using [klibc](<https://git.kernel.org/pub/scm/libs/klibc/klibc.git>), which also had the advantage of being developed and maintained by kernel developers but, while klibc was much more modern, portable, and closer to my needs, it still had the same inconvenience of requiring to be built separately before being linked into the final executable. 

It then started to appear obvious that, if the preinit code had so little dependency on the C library, it could make sense to just define the system calls directly and be done with it. Thus, in January 2017, some system-call definitions were moved to macros. With this new ability to build without relying on any C library at all, a natural-sounding name quickly emerged for this project: "nolibc". 

The first focus was on being able to build existing code as seamlessly as possible, with either nolibc or a regular C library, because ifdefs in the code are painful to deal with. This includes the ability to bypass the "include" block when nolibc was already included on the command line, and not to depend on any external files that would require build-process updates. Because of this, it was decided that only macros and static definitions would be used, which imposes some limitations that we'll cover below. 

One nice advantage for quick tests is that it became possible to include `nolibc.h` directly from the compiler's command line, and to rely on the `NOLIBC` definition coming from this file to avoid including the normal C-library headers with a construct like: 

```
#ifndef NOLIBC
        #include <stdint.h>
        #include <stdio.h>
        /* ... */
        #endif
```

This allows a program to be built with a regular C library on architectures lacking nolibc support while making it easy to switch to nolibc when it is available. This is how rcutorture currently uses it. 

Another point to note is that, while the resulting binary is always statically linked with nolibc (hence it is self-contained and doesn't require any shared library to be installed), it is still usually smaller than with glibc, with either static or dynamic linking: 

```
$ size init-glibc-static init-glibc init-nolibc
        text    data     bss     dec     hex  filename
        707520  22432   26848  756800   b8c40 init-glibc-static
         16579    848   19200   36627    8f13 init-glibc
         13398      0   23016   36414    8e3e init-nolibc
```

Given that this binary is present in all my kernels, my immediate next focus was supporting the various architectures that I was routinely using: i386, x86_64, armv7, aarch64, and mips. This was addressed by providing architecture-specific setup code for the platforms that I could test. It appeared that a few system calls differ between architectures (e.g. `select()` or `poll()` variants exist on some architectures), and even some structures can differ, like the `struct stat` passed to the `stat()` system call. It was thus decided that the required ifdefs would all be inside the nolibc code, with the occasional wrapper used to emulate generic calls, in order to preserve the ease of use. 

#### The painful `errno`

A last challenging point was the handling of `errno`. In the first attempt, the preinit loader used a negative system-call return value to carry an error-return code. This was convenient, but doing so ruins portability because POSIX-compliant programs have to retrieve the error code from the global `errno` variable, and only expect system calls to return -1 on error. (I personally think that this original design is a mistake that complicates everything but it's not going to change and we have to adapt to it). 

The problem with `errno` is that it's expected to be a global variable, which implies that some code has to be linked between nolibc and the rest of the program. That would remove a significant part of the ease of use, so a trade-off was found: since the vast majority of programs using nolibc will consist of a single file, let's just declare `errno` static. It will be visible only from the files that include nolibc, and will even be eliminated by the compiler if it's never used. 

The downside of this approach is that, if a program is made of multiple files, each of them will see a different `errno`. The need to support multi-file programs is rare with nolibc, and programs relying on a value of `errno` collected from another file are even less common, so this was considered as an acceptable trade-off. For programs that don't care and really need to be the smallest possible, it is possible to remove all assignments to `errno` by defining `NOLIBC_IGNORE_ERRNO`. 

#### A proposal

Base on the elements above, it looked like nolibc was the perfect fit for rcutorture. I sent [the proposal](<https://lwn.net/ml/linux-kernel/20180823190657.GA12057@1wt.eu/>), showing what it could achieve: a version of the `sleep` program in a 664-byte binary. (Note that, since then, some versions of binutils have started to add a section called "`.note.gnu.property`", which stores information about the program's use of hardening extensions like [shadow stacks](<https://lwn.net/Articles/885220/>), that pushes the code 4KB apart). 

McKenney expressed interest in this approach, so we tried together to port his existing program to nolibc, which resulted in faster builds and much smaller images for his tests (which is particularly interesting in certain situations, like when they're booting from the network via TFTP for example). With this change, rcutorture automatically detects if nolibc supports the architecture, and uses it in such cases, otherwise falls back to the regular static build. 

The last problem I was seeing is that, these days, I'm more than full-time on [HAProxy](<http://www.haproxy.org/>) and, while I continue to quickly glance over all linux-kernel messages that land in my inbox, it has become extremely difficult for me to reserve time on certain schedules, such as being ready for merge windows. I didn't want to become a bottleneck for the project, should it be merged into the kernel. McKenney offered to host it as part of his -rcu tree and to take care of merge windows; that sounded like a great deal for both of us and that initial work was quickly [submitted](<https://lwn.net/ml/linux-kernel/20181111200127.GA9511@linux.ibm.com/>) for inclusion. 

#### Contributions and improvements

Since the project was merged, there have been several discussions about how to improve it for other use cases, and how to better organize it. Ingo Molnar [suggested](<https://lwn.net/ml/linux-kernel/20181204080837.GA67285@gmail.com/>) pulling it out of the RCU subdirectory to make it easier to use for other projects. The code was duly moved under `tools/`. 

Support for RISC V came in 5.2 by Pranith Kumar; S390 support was posted by Sven Schnelle during 6.2-rc and was merged for a future version. Ammar Faizi [found some interesting ABI issues](<https://lwn.net/ml/linux-kernel/20211011040344.437264-1-ammar.faizi@students.amikom.ac.id/>) and inconsistencies that were [addressed in 5.17](<https://git.kernel.org/linus/937ed91c7122>); that made Borislav Petkov [dig into glibc](<https://lwn.net/ml/linux-kernel/YWXwQ2P0M0uzHo0o@zn.tnic/>) and the [x86 psABI](<https://gitlab.com/x86-psABIs>) (processor specific ABI — basically the calling convention that makes sure that all programs built for a given platform are interoperable regardless of compiler and libraries used). After this work, the psABI spec [was updated](<https://gitlab.com/x86-psABIs/x86-64-ABI/-/merge_requests/25>) to reflect what glibc actually does. 

At this point, it can be said that the project has taken off, as it even received [a tiny dynamic-memory allocator](<https://lwn.net/ml/linux-kernel/20220322102115.186179-1-ammarfaizi2@gnuweeb.org/>) that is sufficient to run `strdup()` to keep a copy of a command-line argument, for example. Signal handling is also [currently being discussed](<https://lwn.net/ml/linux-kernel/20221222035134.3467659-1-ammar.faizi@intel.com/>). 

#### Splitting into multiple files

It appeared obvious that the single-file approach wasn't the most convenient for contributors, and that having to ifdef-out regular include files to help with portability was a bit cumbersome. One challenge was to preserve the convenient way of building by including `nolibc.h` directly from the source directory when the compiler provides its own kernel headers, and to provide an installable, architecture-specific layout. This problem was addressed in 5.19 by having this `nolibc.h` file continue to include all other ones; it also defines `NOLIBC`, which is used by older programs to avoid loading the standard system headers. 

Now the file layout looks like this: 

```
-+- nolibc.h      (includes all other ones)
         +- arch-$ARCH.h  (only in the source tree)
         +- arch.h        (wrapper or one of the above)
         +- ctype.h
         +- errno.h
         +- signal.h
         +- std.h
         +- stdio.h
         +- stdlib.h
         +- string.h
         +- sys.h
         +- time.h
         +- types.h
         +- unistd.h
```

The `arch.h` file in the source tree checks the target architecture and includes the corresponding file. In the installation tree, it is simply replaced by one of these files. In the source tree, it's only referenced by `nolibc.h`. This new approach was already confirmed to be easier to deal with, as [the s390 patches](<https://lwn.net/ml/linux-kernel/20221209141939.3634586-1-svens@linux.ibm.com/>) touched few files. This clean patch set may serve as an example of how to bring support for a new architecture to nolibc. Basically all that is needed is to add a new arch-specific header file containing the assembly code, referencing it in the `arch.h` file, and possibly adjusting some of the existing system calls to take care of peculiarities of the new architecture. Then the kernel self tests and rcutorture should be updated (see below). 

#### Two modes of operation

The kernel defines and uses various constants and structures that user space needs to know; these include system-call numbers, `ioctl()` numbers, structures like `struct stat`, and so on. This is known as the UAPI, for "user-space API", and it is exposed via kernel headers. These headers are needed for nolibc to be able to communicate properly with the kernel. 

There are now two main ways to connect nolibc and the UAPI headers: 

  * The quick way, which works with a libc-enabled toolchain without requiring any installation. In this case, the UAPI headers found under `asm/` and `linux/` are provided by the toolchain (possibly the native one). This mode remains compatible with programs that are built directly from the source tree using "`-include $TOPDIR/tools/include/nolibc/nolibc.h`". 
  * The clean way that is compatible with bare-metal toolchains such as [the ones found on kernel.org](<https://mirrors.edge.kernel.org/pub/tools/crosstool/>), but which requires an installation to gain access to the UAPI headers. This is done with "`make headers_install`". By combining this with the nolibc headers, we get a small, portable system root that is compatible with a bare-metal compiler for one architecture and enables building a simple source file into a working executable. This is the preferred mode of operation for the kernel self tests, as the target architecture is known and fixed. 

#### Tests

The addition of new architectures showed the urgency of developing some self tests that could validate that the various system calls are properly implemented. One feature of system calls is that, for a single case of success, there are often multiple possible causes of failure, leading to different `errno` values. While it is not always easy to test them all, a reasonable framework was designed, making the addition of new tests, as much as possible, burden-free. 

Tests are classified by categories — "syscall" (system calls) and "stdlib" (standard C library functions) only for now — and are simply numbered in each category. All the effort consists of providing sufficient macros to validate the most common cases. The code is not pretty for certain tests, but the goal is achieved, and the vast majority of them are easy to add. 

When the test program is executed, it loops over the tests of the selected category and simply prints the test number, a symbolic name, and "OK" or "FAIL" for each of them depending on the outcome of the test. For example, here are two tests for `chdir()`, one expecting a success and the other a failure: 

```
CASE_TEST(chdir_dot);    EXPECT_SYSZR(1, chdir(".")); break;
        CASE_TEST(chdir_blah);   EXPECT_SYSER(1, chdir("/blah"), -1, ENOENT); break;
```

Some tests may fail if certain configuration options are not enabled, or if the program is not run with sufficient permissions. This can be overcome by enumerating the tests to run or avoid via the `NOLIBC_TEST` environment variable. The goal here is to make it simple to adjust the tests to be run with a boot-loader command line. The output consists of OK/FAIL results for each test and a count of total errors. 

The Makefile in `tools/testing/selftests/nolibc` takes care of runing the tests, including installing nolibc for the current architecture, putting it into an initramfs, building a kernel equipped with it, and possibly starting the kernel under QEMU. All architecture-specific settings are set with variables; these include the defconfig name, kernel image name, QEMU command line, etc. 

The various steps remain separated so that the makefile can be used from a script that would, for example, copy the kernel to a remote machine or to a TFTP server. 

```
$ make -C tools/testing/selftests/nolibc
        Supported targets under selftests/nolibc:
        all          call the "run" target below
        help         this help
        sysroot      create the nolibc sysroot here (uses $ARCH)
        nolibc-test  build the executable (uses $CC and $CROSS_COMPILE)
        initramfs    prepare the initramfs with nolibc-test
        defconfig    create a fresh new default config (uses $ARCH)
        kernel       (re)build the kernel with the initramfs (uses $ARCH)
        run          runs the kernel in QEMU after building it (uses $ARCH, $TEST)
        rerun        runs a previously prebuilt kernel in QEMU (uses $ARCH, $TEST)
        clean        clean the sysroot, initramfs, build and output files
        (...)
```

It is also possible to just create a binary for the local system or a specific architecture without building a kernel (useful when creating new tests). 

One delicate aspect of the self tests is that, if they need to build a kernel, they have to be careful about passing all required options to that kernel. It looked desirable here to avoid confusion by using the same variable names as the kernel (`ARCH`, `CROSS_COMPILE`, etc.) in order to limit the risk of discrepancies between the kernel that is being built and the executable that is put inside. 

As there is no easy way to call the tests from the main makefile, it is not possible to inherit common settings like kernel image name, defconfig, or build options. This was addressed by enumerating all required variables by architecture, and this appears to be the most maintainable. 

While the tests were initially created in the hope of finding bugs in nolibc's system-call implementations (and a few were indeed found), they also proved useful to find bugs in its standard library functions (e.g. the `memcmp()` and `fd_set` implementations were wrong). It is possible that they might, one day, become useful to detect kernel regressions affecting system calls. It's obviously too early for this, as the tests are quite naive and more complete tests already exists in other projects like the [Linux Test Project](<https://github.com/linux-test-project/ltp>). A more likely situation would be for a new system call being developed with its test in parallel, and the test being used to debug the system call. 

#### Current state, limitations, and future plans

The current state is that small programs can be written, built on the fly, and used. Threads are not implemented, which will limit some of the tests, unless the code uses the `clone()` system call and does not rely on thread-local storage or thread cancellation. Mark Brown managed to implement [some tests for TPIDR2 on arm64](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/testing/selftests/arm64/abi/tpidr2.c>) that use threads managed directly inside the test. 

Many system calls, including the network-oriented calls and trivial ones like `getuid()` that nobody has needed, are still not implemented, and the `string.h` and `stdlib.h` functions are still limited, but it is expected that most kernel developers will be able to implement what they are missing when needed, so this is not really a problem. 

The lack of data storage remains a growing concern, first for `errno`, then for the `environ` variable, and finally also for [the auxiliary vector](<https://lwn.net/Articles/519085/>) (AUXV, which contains some settings like the page size). While all of them could easily be recovered by the program from `main()`, such limitations were making it more difficult to implement new extensions, and a patch set addressing this was [recently submitted](<https://lwn.net/ml/linux-kernel/20230109084208.27355-1-w@1wt.eu/>). 

McKenney and I had discussions about having a more interactive tool, that would allow running a wider variety of tests that could all be packaged together. This tool might include a tiny shell and possibly be able to follow a certain test sequence. But it's unclear yet if that should be part of a low-level self-testing project or if, instead, the focus should remain on making it easier to create new tests that could be orchestrated by other test suites. 

#### A tandem team on the project

I don't know if similar "tandem" teams are common in other subsystems, but I must admit that the way the maintenance effort was split between McKenney and myself is efficient. My time on the nolibc project is dedicated to contribution reviews, bug fixes, and improvements — not much more than what it was before its inclusion into the kernel, thanks to McKenney taking care of everything related to inclusion in his -rcu tree, meeting deadlines, and sending pull requests. For contributors, there may be a perceived small increase in time between the moment a contribution is published and when it appears in a future kernel, if I have to adjust it before approving it, but it would be worse if I had to deal with it by myself. 

It took some time and effort to adapt and write the testing infrastructure, but now it's mostly regular maintenance. It seems to me like we're significantly limiting the amount of processing overhead caused by the project on our respective sides, which allows it to be usable in the kernel at a low cost. For McKenney, the benefit is a reduced burden for his testing tools like rcutorture. For me, getting feedback, bug reports, fixes, and improvements for a project that is now more exposed than it used to be, is appreciated.
