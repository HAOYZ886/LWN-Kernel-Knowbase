---
title: Toward the unification of kselftests and KUnit
url: https://lwn.net/Articles/1029077/
date: "July 8, 2025"
category: "Development tools-Testing"
author: "By Jonathan Corbet July 8, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 8, 2025

The kernel project, for many years, lacked a formal testing setup; it was often joked that testing was the project's main reason for keeping users around. While many types of kernel testing can only be done in the presence of specific hardware, there are other parts of the kernel that could be more widely tested. Over time, though, the kernel has gained two separate testing frameworks and a growing body of automated tests to go with them. These two frameworks — kselftests and KUnit — take different approaches to the testing problem; now [this patch series](<https://lwn.net/ml/all/20250626-kunit-kselftests-v4-0-48760534fef5@linutronix.de>) from Thomas Weißschuh aims to bring them together. 

#### Kselftests and KUnit

[Kselftests](<https://docs.kernel.org/dev-tools/kselftest.html>) was first [added by Frederic Weisbecker](<https://git.kernel.org/linus/274343ad3e63>) in 2012, with [the first test](<https://git.kernel.org/linus/85bbddc37b2b>) being focused on the handling of breakpoints on the x86 architecture. These self tests run in user space, exercising the normal kernel system-call interface. Over the years, kselftests has grown a test-harness structure based around the [Test Anything Protocol (TAP)](<https://testanything.org/>) and a set of functions, macros, and makefile support for the creation of tests. 

In current kernels, the [kselftests directory](<https://elixir.bootlin.com/linux/v6.15.5/source/tools/testing/selftests>) has over 100 subdirectories, each containing tests for a specific subsystem. There are tests focused on system calls, architecture support, sysctl knobs, kernel behavior (such as [the sealing of system mappings](<https://lwn.net/Articles/1006375/>)), `/proc` files, and a handful of device drivers, among other things. While this set of tests is not (and probably can never be) complete, a successful kselftests run is enough to give confidence that significant parts of the kernel are working as expected. 

[KUnit](<https://docs.kernel.org/dev-tools/kunit/index.html>) is a different beast; its tests are built as kernel modules and run within the kernel itself. That gives KUnit tests the ability to verify the operation of individual kernel functions that are not reachable from user space. KUnit tests can be loaded into a running kernel and run at any time; they can also be built into the kernel and run automatically at every boot. KUnit, too, provides a set of supporting functions to make tests easier; it is built around a version of TAP called [KTAP](<https://docs.kernel.org/dev-tools/ktap.html>). 

KUnit was [added in 2019](<https://git.kernel.org/linus/6ebf5866f2e8>) by Felix Guo. It, too, has accumulated a growing set of tests over the years; the DRM (graphics) subsystem has quite a few tests, and there is a significant set of tests covering the generic support routines found in the kernel's `lib/` directory. See, for example, [this set of hash-table tests](<https://elixir.bootlin.com/linux/v6.15.5/source/lib/tests/hashtable_test.c>). 

The kernel's two test suites are aimed at different aspects of the testing problem, so it is not entirely surprising that they ended up with different designs. Kselftests pokes at the kernel from the outside, ensuring that its user-space ABI behaves as expected, while KUnit works on the inside, testing components that are not reachable from user space. This separation has allowed each test suite to grow within its targeted space, but it has also led to some occasional frustrations. 

#### Perhaps the twain shall meet

Weißschuh's work appears to be driven by some specific needs felt by embedded developers. The KUnit tests are relatively easy to run on a new embedded system; they can be built into the kernel that is run on those systems, and require no extra support there. The kselftests tests, instead, need a fully working user space on the target system to run. Both that user space and the tests themselves must be loaded onto the target separately from the kernel, making the testing task harder in general. 

The solution that Weißschuh is pursuing is to integrate the two test suites and, in particular, to make it possible to build the kselftests into the kernel and run them from the KUnit framework. A successful solution would allow a kernel and a full set of tests to be loaded onto a target system as a single binary, easing the process of getting the kernel into a fully working state on a new system. There are, naturally, a few obstacles that need to be overcome to get there. 

Kselftests are designed to be run as standalone programs, not as functions within kernel modules like KUnit tests. Weißschuh's series continues to build those tests separately, but the resulting binaries are linked into the new `kunit-uapi` module, which then runs them in a separate kernel thread. This code is based on the user-mode-helper functionality that was [first introduced as part of bpfilter](<https://lwn.net/Articles/755919/>), though a number of changes needed to be made. 

A user-space program expects to have a C library available to it; that is part of the user-space setup that Weißschuh is trying to avoid having to do. Fortunately, since the 5.1 release, the kernel happens to have its own minimal C-library implementation in the form of [nolibc](<https://lwn.net/Articles/920158/>). It is not a complete implementation, but it contains enough support to run much of the kselftest suite. The building of those tests, though, had to be modified to use nolibc rather than the system's C library. 

The `kunit-uapi` module has to do the rest of the work of creating a sufficient environment for each kselftest, as well as actually running the tests. That requires access to functionality that the kernel does not currently export to modules — kernel functions like [`kernel_execve()`](<https://elixir.bootlin.com/linux/v6.15.5/source/fs/exec.c#L1977>), [`replace_fd()`](<https://elixir.bootlin.com/linux/v6.15.5/source/fs/file.c#L1303>), [`create_pipe_files()`](<https://elixir.bootlin.com/linux/v6.15.5/source/fs/pipe.c#L925>), and [`do_exit()`](<https://elixir.bootlin.com/linux/v6.15.5/source/kernel/exit.c#L891>). The series includes [a patch](<https://lwn.net/ml/all/20250626-kunit-kselftests-v4-6-48760534fef5@linutronix.de>) exporting those symbols, among others. The export is limited to the `kunit-uapi` module using the new [`EXPORT_SYMBOL_GPL_FOR_MODULES()`](<https://docs.kernel.org/next/core-api/symbol-namespaces.html#using-the-export-symbol-gpl-for-modules-macro>) macro that was just [added for 6.16](<https://git.kernel.org/linus/707f853d7fa3>); it is the current form of the [restricted-namespace feature](<https://lwn.net/Articles/998221/>) that was first proposed in 2024. 

The newly exported functions are used to set up the standard input, output, and error streams for each test (the input is, for all practical purposes, set to `/dev/null`), mount `/proc`, run the test itself, and clean up afterward. The TAP output from the tests is passed back into the KUnit framework to be reported with the rest of the test results. 

All of this work sets the stage for packaging the existing kselftests, but stops short of that goal. Instead, the only tests enabled at the end of the series are [a simple example test](<https://lwn.net/ml/all/20250626-kunit-kselftests-v4-13-48760534fef5@linutronix.de>) and [a test verifying the `/proc` mount](<https://lwn.net/ml/all/20250626-kunit-kselftests-v4-15-48760534fef5@linutronix.de>). Bringing the existing tests in will require [adding a bit of glue](<https://lwn.net/ml/all/20250708073940-c2e9ee11-549b-4ef0-a480-942d86821f41@linutronix.de>) for each, causing it to be embedded in the loadable module and run at the right time. A more automated way of incorporating the tests is on the wishlist for the future, but does not exist now. 

The incorporation of the actual tests may be waiting for a consensus on the surrounding framework. As of this writing, the series is in its fourth revision, and the most significant concerns would appear to have been addressed; the most recent comments are mostly focused on relatively small issues. Assuming that the biggest problems nearly been overcome, the core framework may find its way into the mainline relatively quickly; the job of integrating all of the actual tests will likely be next.
