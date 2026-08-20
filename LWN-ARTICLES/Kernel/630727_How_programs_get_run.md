---
title: How programs get run
url: https://lwn.net/Articles/630727/
date: "January 28, 2015"
category: exec
author: "January 28, 2015 This article was contributed by David Drysdale"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

January 28, 2015

This article was contributed by David Drysdale

This is the first in pair of articles that describe how the kernel runs programs: what happens under the covers when a user program invokes the [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>) system call? I recently worked on the implementation of a new [`execveat()`](<https://man7.org/linux/man-pages/man2/execveat.2.html>) system call, which is a close variant of `execve()` that allows the caller to specify the invoked program by a combination of file descriptor and path, as with other `*at()` system calls. (This will, in turn, enable an implementation of the [`fexecve()`](<https://man7.org/linux/man-pages/man3/fexecve.3.html>) library function that doesn't [rely on](<https://man7.org/linux/man-pages/man3/fexecve.3.html#NOTES>) access to the `/proc` filesystem, which is important for sandboxed environments such as [Capsicum](<https://lwn.net/Articles/604015/>).) 

Along the way, I explored the existing `execve()` implementation, and so these articles present the details of that functionality. In this one, we'll focus on the general mechanisms that the kernel uses for program invocation, which allow for different program formats; the second article will focus on the details of running ELF binaries. 

#### The view from user space

Before diving into the kernel, we'll start by exploring the behavior of program execution from user space (there's also a good description of this behavior in chapter 27 of [_The Linux Programming Interface_](<https://man7.org/tlpi/index.html>)). For Linux versions up to and including 3.18, the only system call that invokes a new program is `execve()`, which has the following prototype: `

```
int execve(const char *filename, char *const argv[], char *const envp[]);
```

`

The `filename` argument specifies the program to be executed, and the `argv` and `envp` arguments are NULL-terminated lists that specify the command line arguments and environment variables for the new program. A simple [skeleton driver program (`do_execve.c`)](<https://lwn.net/Articles/630754/#do_execve>) allows us to explore how this behaves, by feeding in `"zero"`, `"one"`, `"two"` as arguments and `"ENVVAR1=1"`, `"ENVVAR2=2"` as environment variables. To see the result in the invoked program, we use [another simple program (`show_info.c`)](<https://lwn.net/Articles/630754/#show_info>) that just outputs its command-line arguments (`argv`) and environment ([`environ`](<https://man7.org/linux/man-pages/man7/environ.7.html>)). 

Putting these together gives the expected result -- the arguments and environment are passed through to the invoked program. Notice, though, that the `argv[0]` for the invoked binary is just the value specified by the caller of `execve()`; having the program's name in `argv[0]` isn't a convention that's required or policed by `execve()` itself, at least for binaries. `

```
% ./do_execve ./show_info
        argv[0] = 'zero'
        argv[1] = 'one'
        argv[2] = 'two'
        ENVVAR1=1
        ENVVAR2=2
```

`

Things change slightly when the program being invoked is a script rather than a binary program. To explore this, we use a [shell script equivalent (`show_info.sh`)](<https://lwn.net/Articles/630754/#show_info.sh>) of our environment-outputting program; putting this together with the original program that invokes `execve()`, we see a couple of differences: 

`

```
% ./do_execve ./show_info.sh
        $0 = './show_info.sh'
        $1 = 'one'
        $2 = 'two'
        ENVVAR1=1
        ENVVAR2=2
        PWD=/home/drysdale/src/lwn/exec
```

`

First, the environment has gained an extra `PWD` value, indicating the current directory. Secondly, the initial argument to the script is now the script filename, rather than the `"zero"` value that the invoker specified. A further experiment reveals that the `/bin/sh` script interpreter added the `PWD` environment variable, but the kernel itself modified the arguments: 

`

```
% cat ./wrapper
        #!./show_info
        
        % ./do_execve ./wrapper
        argv[0] = './show_info'
        argv[1] = './wrapper'
        argv[2] = 'one'
        argv[3] = 'two'
        ENVVAR1=1
        ENVVAR2=2
```

`

More specifically, the kernel has removed the first (`"zero"`) argument and replaced it with two arguments -- the name of the script interpreter program (taken from the first line of the script) and the name of the invoked file (which holds the script text). If the first line of the script also includes command-line arguments for the interpreter (for example, `awk` needs an `-f` option to treat its input as a filename rather than script text), a third extra argument is also inserted, holding all of the extra options: 

`

```
% cat ./wrapper_args
        #!./show_info -a -b -c
    
        % ./do_execve ./wrapper_args
        argv[0] = './show_info'
        argv[1] = '-a -b -c'
        argv[2] = './wrapper_args'
        argv[3] = 'one'
        argv[4] = 'two'
        ENVVAR1=1
        ENVVAR2=2
```

`

Up to a point, we can also repeat this pop-one, push-two alteration of the arguments, by invoking scripts that wrap scripts and so on; each such alteration effectively pushes the wrapper script name in at `argv[1]`: 

`

```
argv[0]:  'zero'=>'./wrapper4'=>'./wrapper3'=>'./wrapper2'=>'./wrapper' =>'./show_info'
        argv[1]:  'one'   './wrapper5'  './wrapper4'  './wrapper3'  './wrapper2'  './wrapper'
        argv[2]:  'two'   'one'         './wrapper5'  './wrapper4'  './wrapper3'  './wrapper2'
        argv[3]:          'two'         'one'         './wrapper5'  './wrapper4'  './wrapper3'
        argv[4]:                        'two'         'one'         './wrapper5'  './wrapper4'
        argv[5]:                                      'two'         'one'         './wrapper5'
        argv[6]:                                                    'two'         'one'
        argv[7]:                                                                  'two'
```

`

However, this doesn't continue forever -- once there are too many levels of wrappers, the process fails with `ELOOP`: 

`

```
% ./do_execve ./wrapper6
        Failed to execute './wrapper6', Too many levels of symbolic links
```

`

#### Into the kernel: `struct linux_binprm`

Now we move into kernel space and begin delving into the code that implements the `execve()` system call. A [previous article](<https://lwn.net/Articles/604515/>) explored the general system call machinery (and the special wrinkles needed for `execve()`), so we can pick up the story at the [`do_execve_common()`](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1430>) function in `fs/exec.c`. The main purpose of the code in this function is to [build](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1463>) a new [`struct linux_binprm`](<https://elixir.bootlin.com/linux/v3.18/source/include/linux/binfmts.h#L14>) instance that describes the current program invocation operation. In the structure: 

  * The `file` field is [set](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1481>) to a freshly opened `struct file` for the program being invoked; this allows the kernel to read the file contents and decide how to handle the file.
  * The `filename` and `interp` fields are [both set](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1482>) to the name of the file holding the program; we'll see later why there are two distinct fields here.
  * The `bprm_mm_init()` function [allocates](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L366>) and [sets up](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L268>) the associated [`struct mm_struct`](<https://elixir.bootlin.com/linux/v3.18/source/include/linux/mm_types.h#L345>) and [`struct vm_area_struct`](<https://elixir.bootlin.com/linux/v3.18/source/include/linux/mm_types.h#L247>) data structures in preparation for managing the virtual memory of the new program. In particular, the new program's virtual memory ends at the [highest possible address](<https://elixir.bootlin.com/linux/v3.18/source/arch/x86/include/asm/processor.h#L912>) for the architecture; its stack will grow downward from there.
  * The `p` field is [set to point at the end of memory space](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L281>) for the new program, but leaves space for a NULL pointer as an end marker for the stack. The value of `p` will be updated (downward) as more information is added to the new program's stack.
  * The `argc` and `envc` fields [are set](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1488>) to hold the counts of arguments and environment values so that this information can be propagated to the new program later in the invocation process.
  * The `unsafe` field is [set up](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1217>) to hold a bitmask of reasons why the program execution might not be safe; for example, if the process is being traced with [`ptrace()`](<https://man7.org/linux/man-pages/man2/ptrace.2.html>) or has the [`PR_SET_NO_NEW_PRIVS` bit](<https://lwn.net/Articles/475678/>) set. The Linux Security Module (LSM) may subsequently use this information to deny the program execution operation.
  * The `cred` field is a [separately allocated](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1147>) object of type [`struct cred`](<https://elixir.bootlin.com/linux/v3.18/source/include/linux/cred.h#L102>) that holds information about the credentials for the new program. These are generally [inherited](<https://elixir.bootlin.com/linux/v3.18/source/kernel/cred.c#L250>) from the process that called `execve()`, but are updated to allow for [setuid](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1274>) / [setgid](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1280>) bits and other complications. The presence of setuid/setgid bits also disallows a collection of [compatibility features](<https://elixir.bootlin.com/linux/v3.18/source/include/uapi/linux/personality.h#L30>) because they have an adverse effect on security; the `per_clear` field records the bits in the process's [personality](<https://man7.org/linux/man-pages/man2/personality.2.html>) that will be cleared later. 
  * The `security` field allows an LSM to store LSM-specific information with the `linux_binprm`; the LSM is notified via a call to [`security_bprm_set_creds()`](<https://elixir.bootlin.com/linux/v3.18/source/security/security.c#L214>) and the [`bprm_set_creds`](<https://elixir.bootlin.com/linux/v3.18/source/include/linux/security.h#L202>) LSM hook. The [default implementation](<https://elixir.bootlin.com/linux/v3.18/source/include/linux/security.h#L1996>) of this hook updates the new program's [Linux capabilities](<https://man7.org/linux/man-pages/man7/capabilities.7.html>) to [allow for the file capabilities](<https://elixir.bootlin.com/linux/v3.18/source/security/commoncap.c#L475>) of the program file; other LSM implementations chain this behavior into their own implementations of the hook (e.g. [Smack](<https://elixir.bootlin.com/linux/v3.18/source/security/smack/smack_lsm.c#L676>), [SELinux](<https://elixir.bootlin.com/linux/v3.18/source/security/selinux/hooks.c#L2144>)).
  * The `buf` scratch space is [filled](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1299>) with the first chunk ([128 bytes](<https://elixir.bootlin.com/linux/v3.18/source/include/uapi/linux/binfmts.h#L18>)) of data from the program file. This data will be used later to detect the binary format so it can be processed appropriately.

The parts of this setup process that depend on the particular file that's being executed are performed in an inner [`prepare_binprm()`](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1253>) function; this function can be called again later to update those fields if a different file (e.g. a script interpreter) is actually run. 

Finally, information about the program invocation is copied into the top of new program's stack, using the local [`copy_strings()`](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L453>) and [`copy_strings_kernel()`](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L544>) utility functions. First, the program filename is pushed to the stack (and its location is [saved](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1504>) in the `exec` field of the `linux_bprm` instance), followed by all of the environment values, then by all of the arguments. At the end of this process, the stack looks like: 

`

```
---------Memory limit---------
        NULL pointer
        program_filename string
        envp[envc-1] string
        ...
        envp[1] string
        envp[0] string
        argv[argc-1] string
        ...
        argv[1] string
        argv[0] string
```

`

#### Binary format handler iteration: `struct linux_binfmt`

With a complete `struct linux_binprm` in hand, the real business of program execution is performed in [`exec_binprm()`](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1405>) and (more importantly) [`search_binary_handler()`](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1352>). This code [iterates](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1369>) over a list of [`struct linux_binfmt`](<https://elixir.bootlin.com/linux/v3.18/source/include/linux/binfmts.h#L70>) objects, each of which provides a handler for a particular format of binary programs. A binary handler could potentially be defined in a kernel module, so the code calls [`try_module_get()`](<https://elixir.bootlin.com/linux/v3.18/source/kernel/module.c#L945>) for each format to ensure the relevant code can't be unloaded by another task while it's being used here. 

For each `struct linux_binfmt` handler object, the `load_binary()` function pointer is called, passing in the `linux_binprm` object. If the handler code supports the binary format, it does whatever is needed to prepare the program for execution and returns success (>= 0). Otherwise, the handler returns a failure code (< 0) and iteration continues with the next handler. 

Execution of a particular program may itself rely on execution of a different program; the obvious example is executable scripts, which need to invoke the script interpreter. To cope with this, the `search_binary_handler()` code can be called recursively, re-using the `struct linux_binprm` object. However, recursion depth is [limited](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1358>) to prevent infinite recursion, giving the `ELOOP` error behavior seen earlier. 

The system's LSM also gets a say in the operation; before the iteration over binary formats starts, the [`bprm_check_security`](<https://elixir.bootlin.com/linux/v3.18/source/include/linux/security.h#L214>) LSM hook is [triggered](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1362>), allowing the LSM to make a decision on whether to allow the operation. To do so, it may use the state it stored in the `linux_binprm.security` field earlier. 

At the end of the iteration, if no formats that can handle the program have been found (and the program appears to be binary rather than text, at least [according to the first four bytes](<https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L1392>)), then the code will also attempt to load a module named `"binfmt-_XXXX_ "`, where XXXX is the hex value of bytes three and four in the program file. This is an [old mechanism](<https://www.tldp.org/HOWTO/Kerneld/configuration.html#BINFMT>) (added in 1996 for Linux 1.3.57) to allow for a more dynamic way of associating binary format handlers with formats; the more recent `binfmt_misc` mechanism (described below) allows a more flexible way of doing something similar. 

#### Binary formats

So what are the binary formats available in the standard kernel? A search for code that registers instances of `struct linux_binfmt` (via [`register_binfmt()`](<https://elixir.bootlin.com/linux/v3.18/source/include/linux/binfmts.h?v=3.17#L82>) and [`insert_binfmt()`](<https://elixir.bootlin.com/linux/v3.18/source/include/linux/binfmts.h?v=3.17#L87>)) gives us quite a collection of possible formats, all of which are configured and explained in the [`fs/Kconfig.binfmts`](<https://elixir.bootlin.com/linux/v3.18/source/fs/Kconfig.binfmt>) file: 

  * [`binfmt_script.c`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_script.c#L108>): Support for interpreted scripts, starting with a `#!` line.
  * [`binfmt_misc.c`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_misc.c#L729>): Support miscellaneous binary formats, according to runtime configuration.
  * [`binfmt_elf.c`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L2200>): Support for ELF format binaries.
  * [`binfmt_aout.c`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_aout.c#L412>): Support for traditional a.out format binaries.
  * [`binfmt_flat.c`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_flat.c#L945>): Support for [flat format](<https://elinux.org/UClinux_Shared_Library#Shared_FLAT>) binaries.
  * [`binfmt_em86.c`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_em86.c#L102>): Support for Intel ELF binaries running on Alpha machines.
  * [`binfmt_elf_fdpic.c`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf_fdpic.c#L94>): Support for [ELF FDPIC](<https://elinux.org/UClinux_Shared_Library#FDPIC_ELF>) binaries.
  * [`binfmt_som.c`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_som.c#L286>): Support for SOM format binaries (an HP/UX PA-RISC format).

(plus a couple of other architecture-specific formats). 

The next sections will examine the most important of these: interpreted scripts and the "miscellaneous" mechanism for supporting arbitrary formats; the next article will examine the ELF binary format -- which is typically where all program execution ends up. 

#### Script invocation: `binfmt_script.c`

Files that start with the character sequence `#!` (and have the execute bit set) are treated as scripts, handled by the [`fs/binfmt_script.c`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_script.c>) handler. After checking those first two bytes, this code parses the rest of the script-invocation line, splitting it into an interpreter name (everything after `#!` up to the first white space) and possible arguments (everything else up to the end of the line, stripping external white space). 

(One detail to note: back when the `struct linux_binprm` object was created, only the first 128 bytes of the program were retrieved. This means that if the interpreter name and arguments are longer than this, the results will be truncated.) 

With these in hand, the code then [removes `argv[0]`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_script.c#L69>) from the top of the new program's stack (i.e. at the lowest address), and in its place pushes the following, adjusting the `argc` value in the `linux_binprm` object along the way: 

  * the [program name](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_script.c#L72>)
  * (optionally) the [collected interpreter arguments](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_script.c#L76>)
  * the [name of the interpreter program](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_script.c#L80>)

Taken together, this explains the user space behavior we observed at the beginning of the article; our new program's stack is modified to look like: 

`

```
---------Memory limit---------
        NULL pointer
        program_filename string
        envp[envc-1] string
        ...
        envp[1] string
        envp[0] string
        argv[argc-1] string
        ...
        argv[1] string
        program_filename string
        ( interpreter_args )
        interpreter_filename string
```

`

The code also [changes the `interp` value](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_script.c#L83>) in the `linux_binprm` structure so that it references the interpreter filename, rather than the script filename. This explains why the `linux_binprm` structure refers to two strings: one (`interp`) is the program that we currently want to execute, and one is the name (`filename`) that was originally invoked in the `execve()` call. Along similar lines, the `file` field in the `linux_binprm` is also [updated](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_script.c#L94>) to reference the new interpreter program, and the first 128 bytes of its contents [read into](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_script.c#L95>) the `buf` scratch space. 

The script handler code then [recurses](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_script.c#L98>) into `search_binary_handler()` to repeat the whole process for the script interpreter program. If the interpreter is itself a script, then the `interp` value will be changed once again but the `filename` will stay unchanged. 

#### Miscellaneous interpreter detection: `binfmt_misc.c`

We saw previously that early versions of the Linux kernel supported a rough-and-ready way of dynamically adding format support, by hunting for a kernel module with a name containing the early bytes of the binary. That's not particularly convenient -- only searching on a couple of bytes is very limited (compare the [vast range of detection signatures](<https://github.com/file/file/tree/master/magic/Magdir>) that the [`file`](<https://man7.org/linux/man-pages/man1/file.1.html>) command uses) and requiring a kernel module raises the barrier to entry. 

The [miscellaneous binary format handler](<https://elixir.bootlin.com/linux/v3.18/source/Documentation/binfmt_misc.txt>) allows a more flexible and dynamic method of dealing with new formats, by allowing run-time configuration (via a special [filesystem](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_misc.c#L727>) mounted under `/proc/sys/fs/binfmt_misc`) to specify: 

  * How to recognize a supported format, based on filename extension or a magic value at a particular offset. (As with parsing script interpreters, this magic value has to fall within the first 128 bytes of the program file.)
  * The interpreter program to invoke, which will get the program filename passed to it as `argv[1]` (as with script invocation).

A good example of the miscellaneous format handler in use is for [Java files](<https://elixir.bootlin.com/linux/v3.18/source/Documentation/java.txt>): detect `.class` files (based on their [`0xCAFEBABE` prefix](<https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.1>)) or `.jar` files (based on the `.jar` extension) and automatically invoke the JVM executable on them. This will require a wrapper script to provide the relevant command-line arguments, as the miscellaneous configuration doesn't allow arguments to be specified -- which means that the miscellaneous handler will invoke the script handler, which will then invoke the ELF handler for the JVM executable (and which will probably in turn invoke the dynamic linker `ld.so`, although that's a somewhat different story). 

Internally, the [kernel implementation](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_misc.c>) for this format is similar to the handler for script programs described above, except that there is an initial [search for a matching configuration entry](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_misc.c#L85>), and that configuration is used to make some of the details (such as [removing `argv[0]`](<https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_misc.c#L145>)) optional. 

The format handlers for both scripts and miscellaneous formats recurse on to attempt to invoke the interpreter program that is needed for that particular format. This recursion has to end at some point, and on a modern Linux system this is almost always at an ELF binary program -- the subject of the next article -- stay tuned.
