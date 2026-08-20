---
title: Bringing BPF to binfmt_misc
url: https://lwn.net/Articles/1086947/
date: "August 6, 2026"
category: "BPF; Releases-7.3; binfmt misc"
author: "By Jonathan Corbet August 6, 2026"
---

> **For humans, by humans**
> 
> Every article on LWN.net is written for humans, by humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the slop at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 6, 2026

The kernel is able to run a few types of executable files, including native binaries in the ELF format and interpreted programs that begin with the `#!` marker. It also, however, has a mechanism, called [binfmt_misc](<https://docs.kernel.org/admin-guide/binfmt-misc.html>), that can be configured from user space to enable the transparent execution of programs in just about any format. This feature has been relatively static for years, but it seems likely to receive some significant updates in the near future, including the ability to load BPF programs that can decide how to run a given program. 

#### binfmt_misc

In the beginning, Linux used the [a.out format](<https://en.wikipedia.org/wiki/A.out>) for binary executables. Support for the [Executable and Linkable Format (ELF)](<https://en.wikipedia.org/wiki/Executable_and_Linkable_Format>) was added to the 0.99.13 release in 1993, with distribution support (involving upgrades so painful that they still cause post-traumatic stress many years later) arriving around 1995. But, from the earliest days, there was interest in running programs in different formats as well. There were a number of Unix variants available for x86 systems, for example, each with its own binary format. There are languages, such as Java, that compile to their own virtual machines and need a separate interpreter. The kernel _could_ gain support for all of these formats, but it was understood early on that a more general solution was called for. 

That solution is binfmt_misc, which was added for the 2.1.43pre1 development release in 1997. In its current form, binfmt_misc presents a directory under `/proc/sys/fs/binfmt_misc` containing a file called `register`. Writing a string to that file registers a new format and tells the kernel how to recognize executables in that format; recognition can be based on a pattern in the first 256 bytes of the file itself, or on the file's name extension. For example, the documentation states that support for packed DOS applications can be had by writing this string to the `register` file: 

```
:DEXE:M::\x0eDEX::/usr/bin/dosexec:
```

This line causes the kernel to recognized a packed DOS executable by the four-byte sequence `\x0eDEX` at the beginning of the file; it will invoke `/usr/bin/dosexec` to run that executable. As another example, any file with a `.py` extension and execute permission can be executed directly with: 

```
:python:E::py::/usr/bin/python3:
```

After that has been done, `.py` files will be executed by the Python interpreter regardless of whether they contain a `#!` line. Multiple binfmt_misc entries can be made; the first matching entry is selected whenever an interpreter for a program must be found. In a delightful quirk, entries are checked in the opposite order that one might expect; the most recently added entries are checked first. 

With binfmt_misc in place, the kernel developers allowed user space to set up whatever policy it needs for its own special executable formats; there was no longer a need to add dedicated support to the kernel for them. While this feature has seen a slow stream of improvements over the years, it has not seen many fundamental changes since its introduction nearly three decades ago. 

#### Hermetic binaries

The formats defined with binfmt_misc all involve an interpreter that handles the actual execution of the program. Native binaries, instead, run directly on the CPU, but (unless they are statically linked) they still involve an interpreter; in that case, the interpreter is the dynamic linker, which is charged with loading the binary code into memory, loading libraries, and resolving references. The location of that interpreter is, for ELF files, found in the ELF binary itself, in the `PT_INTERP` segment; it is typically something like `/lib64/ld-linux-x86-64.so.2`. The kernel loads the interpreter, then lets it do the rest of the work in user space. 

This scheme has worked well for many years, but there is always somebody who wants to do something different. Back in June, Farid Zakaria [made the case](<https://fzakaria.com/2026/06/21/nix-needs-relocatable-binaries>) that the Nix distribution needs relocatable binaries — binary programs that are fully self-contained (hermetic) and can run regardless of where they are placed in the filesystem hierarchy. A hermetic binary needs, among other things, a specific version of the dynamic linker, which may not be the default one installed on the system where the binary is expected to run. The `PT_INTERP` mechanism can allow for the specification of a different dynamic linker, but it requires an absolute path to that linker. That makes it hard to install the program (and its linker) in an arbitrary location without the need to patch the executable or engage in tricks with [`chroot()`](<https://man7.org/linux/man-pages/man2/chroot.2.html>) or namespaces. As a result, building this type of system is harder than it seems it should be. 

Zakaria's [proposed solution](<https://lwn.net/ml/all/20260622043934.179879-1-farid.m.zakaria@gmail.com/>) to this problem was to allow `PT_INTERP` to specify a path that is relative to the location where the executable is stored. While the kernel community was supportive of end goal (running hermetic binaries), this approach raised a lot of concerns. Kees Cook [worried](<https://lwn.net/ml/all/202606231236.325C882A78@keescook/>) about security problems resulting from manipulating paths in the kernel. Christian Brauner was [more strongly negative](<https://lwn.net/ml/all/20260625-atomkraftgegner-hunger-kursbuch-b452ff2becab@brauner/>), saying that Zakaria's approach was ""ripe for malicious loader injection attacks"" and that the problem should be solved in a different way. Specifically, he [thought](<https://lwn.net/ml/all/20260703-meditation-ratsuchende-moratorium-9ecdf1f3f8bb@brauner/>), the right approach was to augment binfmt_misc to allow the installation of a BPF program to select the appropriate interpreter for a program. 

#### BPF for binfmt_misc

Less than two weeks later, Brauner returned with [a patch set](<https://lwn.net/ml/all/20260707-work-bpf-binfmt_misc-v1-0-74b995c84ec1@kernel.org>) implementing that functionality. At the time, he said that he intended to step away from the work at that point and let Zakaria finish the job. Zakaria duly responded with [patches of his own](<https://lwn.net/ml/all/20260711-binfmt-misc-bpf-v2-v2-0-d6591ceaf207@gmail.com>) a few days later. Brauner, showing how one truly rests while on vacation, continued to develop the idea himself in the meantime, though, returning on July 14 with [an updated patch set](<https://lwn.net/ml/all/20260714-work-bpf-binfmt_misc-v2-0-57b7529c002c@kernel.org>) that would appear to be the form the solution will take in the end. 

There are two components to this work: the BPF interface, and additions to binfmt_misc. On the BPF side, there is a new [struct_ops](<https://lwn.net/Articles/974848/>) interface allowing programs to set themselves up to manage interpreters. A program registers a new operations structure with two callbacks: 

```
struct binfmt_misc_ops {
    	bool (*match)(struct linux_binprm *bprm);
    	int (*load)(struct linux_binprm *bprm);
    	char name[BINFMT_MISC_OPS_NAME_MAX];
        };
```

The `match()` callback receives a pointer to a [`linux_binprm`](<https://elixir.bootlin.com/linux/v7.1.5/source/include/linux/binfmts.h#L15>) structure describing the program that is in the process of being launched; its job is to decide whether the BPF program is the correct one to manage the interpreter in this case. It serves the same function as the tests on specific bytes or a file extension does in the current binfmt_misc implementation. This program is sleepable, and it has the ability to read any part of the executable; unlike existing binfmt_misc tests, it is not limited to the first 256 bytes. 

If the `match()` callback returns a true value, the `load()` callback will be invoked to select the interpreter that is to be used to run this program. There is a new kfunc, `bpf_binprm_set_interp()`, that is used to associate the interpreter with the program. If `match()` has returned true, `load()` must succeed, or the running of the program will fail. 

The `name` field is used to identify this program when configuring binfmt_misc. That configuration is done by writing a string like: 

```
:name:B::::bpf-handler-name:
```

to the `register` file. Here, `name` is the name by which the binfmt_misc entry is known (there will be a subdirectory created under `/proc/sys/fs/binfmt_misc` with that name), while `bpf-handler-name` is the name used when registering the BPF program. Once that registration is complete, the identified BPF program will be invoked whenever the kernel is trying to figure out how to run an executable program. 

This functionality, it seems, is sufficient to satisfy the Nix use case; Zakaria put up [a satisfied blog post](<https://fzakaria.com/2026/07/20/linux-kernel-will-support-origin-sort-of>) that includes an example of the sort of simple BPF program needed to make relocatable binaries work. He pointed out that this feature can also be used to make the interpreter specified in `#!` lines relocatable, solving the problem for the script case as well. 

#### Further steps

One might think that, with a seemingly satisfactory solution, Brauner would go back to his vacation with more enjoyment than before, but that was not to be. Zakaria had pointed out one little problem with the new binfmt_misc: the interpreter _becomes_ the program that is run, while the program the user _thought_ they were running ends up being an argument passed to the interpreter. It is the same effect that one sees with scripts on current systems; a tool like `ps` will show the interpreter as the running program rather than the script. Having `ps` or `top` show a system full of processes all running the dynamic linker might just lead to a complaint or two from users. 

Brauner's answer to that was [a 21-part patch series](<https://lwn.net/ml/all/20260721-work-bpf-binfmt_misc-ptinterp-v2-0-e57866e4ae0f@kernel.org>) adding more binfmt_misc features. One of these is a new "transparent dispatch" mode that can be requested with either a flag (`T`) in the `binfmt_misc` entry or specified by the matched BPF program. When this mode is selected, the kernel behaves as if it were executing the binary directly, but substitutes the interpreter behind the scenes. The argument vector ("argv") and task command string remain as originally passed to [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>). The interpreter will receive an open file descriptor (specified in the `AT_EXECFD` [auxiliary vector](<https://www.man7.org/linux/man-pages/man3/getauxval.3.html>) entry) that it can use to load and run the program in whatever special way it requires. 

There is also a simpler mode that can be selected when the program to run is actually a native binary on the target system. In that case, the "loader substitution" mode, selected with the `L` flag, simply causes the interpreter returned from the BPF program to override whatever `PT_INTERP` was stored in the executable. So, in cases where the only thing that the BPF program must do is figure out where the dynamic linker is to be found (such as in the Nix case), the loader substitution mode is all that is needed. 

Using either mode solves the problem that Zakaria pointed out, causing the executed program to show up under its own name rather than that of the interpreter. 

The story is still not done, though. A BPF program has control over the path name for the interpreter that should run a given program, but it has no say over how that path name will be resolved. That resolution will happen in the namespace where the program was invoked. That can lead to problems if somebody with privileges in that namespace can, by mounting new filesystems, direct the resolution toward a special "interpreter" of their own. This problem is especially acute in cases where setuid programs are being run. Giving a potential attacker the ability to supply their own interpreter does not seem like a path toward pleasing results. 

For normal binfmt_misc entries, where the interpreter is specified in the entry itself, the kernel can be instructed to open the file containing the interpreter and use it for every subsequent invocation. That ensures that the intended interpreter is always used, regardless of the environment in which a program is run. This feature was [added in 2016](<https://lwn.net/Articles/679308/>) as a way of making interpreters available inside containers that are otherwise isolated from the host's filesystem. 

A BPF program, though, can set the interpreter path to anything it wants, making it hard for the kernel to open any specific interpreter ahead of time. Brauner, naturally, has [a patch series](<https://lwn.net/ml/all/20260730-work-binfmt_misc-preopen-v1-0-4a0b0da71f16@kernel.org>) for that problem too. It allows the binfmt_misc configuration to include a set of known interpreters that will be opened by the kernel at configuration time; a BPF program can then choose between them whenever a specific program is to be run. 

There were a couple of difficulties to overcome to enable this mode, though. In current kernels, a binfmt_misc entry must be fully specified with a single write to the `register` file; that entry becomes globally visible immediately after the write completes, so it cannot be left in a half-specified state. There is also a length limit on lines written to `register` that might constrain the number of interpreters that can be specified, especially if their path names are long. The solution to both problems was to create a new "disabled" mode allowing a binfmt_misc entry to be created without it becoming immediately active; multiple lines can then be used to adjust that entry before enabling it. An example of its use is given in the cover letter: 

```
cd /proc/sys/fs/binfmt_misc
        echo ':qemu:B::::qemu_user:D' > register
        echo '+aarch64 /usr/bin/qemu-aarch64' > qemu
        echo '+arm /usr/bin/qemu-arm' > qemu
        echo 1 > qemu
```

This sequence starts by creating a new entry, called `qemu`, that will invoke a BPF program named `qemu_user`; this entry will be represented by a file (also named `qemu`) in the control directory. The `D` at the end of the first `echo` command indicates that the entry is to be initially disabled. The following two lines add two interpreter choices, named `aarch64` and `arm`, each with its own program that will be held open by the kernel. The `echo 1` at the end enables the entry. 

When the BPF program's `load()` callback runs on a matched program, it no longer needs to provide the full path to the interpreter; instead, it just gives the name of one of the preconfigured interpreters. The program will then be run with the given interpreter, again using the same file regardless of which namespace the program was invoked in. 

As a final (so far) step, Brauner tossed in [a short series](<https://lwn.net/ml/all/20260803-work-binfmt_misc-interplimit-v1-0-4a2435500bd9@kernel.org>) setting up a knob to restrict how many interpreter files a given user can hold open. Without that limit, a hostile user running in a user namespace might be able to open enough "interpreter" files to starve the system of resources. 

With the exception of the final resource-limiting series, all of this work is currently in linux-next, so it can be expected to land in the mainline during the 7.3 merge window. There is some documentation of the new features in [Documentation/admin-guide/binfmt_misc.rst](<https://docs.kernel.org/next/admin-guide/binfmt-misc.html>) for the curious. The kernel's ability to transparently invoke binaries in exotic formats is about to become much more flexible; some people will, beyond doubt, do interesting things with it.
