---
title: Designing ELF modules
url: https://lwn.net/Articles/749108/
date: "March 13, 2018"
category: "Modules-ELF modules"
author: "By Jonathan Corbet March 13, 2018"
---

By **Jonathan Corbet**  
March 13, 2018

The [bpfilter proposal](<https://lwn.net/Articles/747551/>) posted in February included a new type of kernel module that would run as a user-space program; its purpose is to parse and translate iptables rules under the kernel's control but in a contained, non-kernel setting. These "ELF modules" were [reposted](<https://lwn.net/Articles/748627/>) for review as a standalone patch set in early March. That review has happened; it is a good example of how community involvement can improve a special-purpose patch and turn it into a more generally useful feature. 

ELF modules look like ordinary kernel modules in a number of ways. They are built from source that is (probably) shipped with the kernel itself, they are compiled to a file ending in `.ko`, and they can be loaded into the kernel with `modprobe`. Rather than containing a real kernel module, though, that `.ko` file holds an ordinary ELF binary, as a user-space program would. When the module is "loaded", a special process resembling a kernel thread is created to run that program in user mode. The program will then provide some sort of service to the kernel that is best not run within the kernel itself. 

In general, the community's reaction to this feature may have been [expressed best](<https://lwn.net/Articles/749113/>) by Greg Kroah-Hartman: ""this is crazy stuff, but I like the idea and have no objection to it overall"". ELF modules give the kernel a controlled way to run user-space helper code, and they make it easy to develop and distribute that code with the kernel itself. That latter aspect, in particular, distinguishes ELF modules from the existing "usermode helper" mechanism, which depends on programs developed and shipped separately from the kernel. It's clear that some developers see uses for this feature beyond the bpfilter subsystem, and would like for those uses to be supported as well. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Beyond rule translation

Consider, for example, one branch of the discussion where Andy Lutomirski [raised concerns](<https://lwn.net/Articles/749115/>) that the current implementation might break systems that load an ELF module during system boot. Alexei Starovoitov, the author of the patches, [responded](<https://lwn.net/Articles/749116/>): ""There is no intent to use umh modules during boot process. This is not a replacement for drivers and kernel modules"". Instead, he said, this feature is aimed at one specific use: converting iptables rules to BPF programs. But some developers, including Kroah-Hartman, are clearly [looking further ahead](<https://lwn.net/Articles/749117/>): 

You are creating a very generic, new, user/kernel api that a whole bunch of people are going to want to use. Let's not hamper the ability for us all to use this right from the beginning please. 

In particular, he sees uses for these modules as a way to implement USB drivers in user space, perhaps bringing some existing user-space drivers into the kernel tree in the process. 

Making ELF modules serve the more general use case may require a number of changes to the patch set. As Linus Torvalds [pointed out](<https://lwn.net/Articles/749118/>), there is a significant difference between standard kernel modules and the current implementation of ELF modules. When the process of loading a standard module completes, that module has registered itself with all of the requisite subsystems and is ready to respond to requests from the kernel or user space. The end of the loading process for an ELF module, though, only indicates that the program in the module has started executing. It may not yet be ready to answer requests or provide services and, should something go wrong in its initialization process, it may crash and never get to that point. 

The answer to this problem (and a couple of others), according to Torvalds, is to make the execution of ELF modules synchronous, in that a `modprobe` invocation would not complete until the process that was started to run the module's code has exited. For short-duration tasks, the final exit status could reflect the success of the operation itself, which is not possible in the current implementation. For a long-running module, the code could fork and return a success status once initialization is complete, giving a clear indication that the module is ready to do its work. 

Some other changes would be required to make ELF modules suitable for other use cases. Currently there is no means of communication between the module and the kernel beyond the standard system calls. If ELF modules are to be used for tasks like driving a new device, there will need to be a way to pass control of that device to the module from the kernel, among other things. A number of these issues could apparently be handled by opening a pipe between the kernel and the module when it is launched and using it for communications between the two. 

A trickier problem may have to do with modules that need some sort of filesystem access to operate. The access itself can be provided, but it can be difficult to write such code in a way that doesn't assume some sort of filesystem layout (the existence and contents of `/dev`, for example) in the underlying system. The kernel tries hard not to impose such policies on user space, and nobody would like to see that change with ELF modules. 

#### Security concerns

Another issue that came up in the conversation is security. Kees Cook [argued](<https://lwn.net/Articles/749119/>) that there were a number of security issues with ELF modules. They run with full privileges regardless of the privilege level of the process that caused them to be loaded, and they run in the root namespace even if they were loaded in response to a request from inside a container. Most of the security concerns have been pushed aside for a simple reason: standard kernel modules run with full privileges inside the kernel itself. Even a process running as root is not as privileged as an normal kernel module, so it is unlikely that adding this feature will make the system less secure, especially if module signing is used to limit the modules that can be loaded. 

One interesting exception did turn up later in the conversation, though. As Torvalds [pointed out](<https://lwn.net/Articles/749122/>), there is a race window between the time that the module signature is checked and when the code is actually loaded into memory and executed; an attacker with the `CAP_SYS_MODULE` capability could exploit this window to replace the code between those two steps. That escalates the ability to run an existing, signed module into the ability to run arbitrary code as root. One way of addressing this issue would be the synchronous behavior described above. The kernel could take control of the file containing the module, marking it as non-writable, for the duration of the module's execution. 

Another possible solution would be to load the code into kernel memory first, perform the check, then execute from that copy of the code. Lutomirski, in a separate part of the discussion, had [suggested](<https://lwn.net/Articles/749125/>) a mechanism where the code would be stored as a binary blob within a standard kernel module; the kernel would then execute the contents of the blob after loading the module. This approach, too, would avoid the race window described above. It would also make the ELF-module functionality work in non-modular kernels (assuming the module is built in, of course) and enable tighter integration with the rest of the kernel. 

The downside of these approaches is that they load the module code into kernel memory, which is not pageable. For tiny modules that would not be a problem, but ELF modules, like other kernel code, seem likely to grow over time. Lutomirski [suggested](<https://lwn.net/Articles/749126/>) that the module code could be backed up by a tmpfs filesystem; Kroah-Hartman [responded](<https://lwn.net/Articles/749127/>) that it would be ""tricky"" but that it could be a good solution. ""Micro-kernel here we come!"" But no such implementation exists now. 

There were few solid conclusions from the discussion, due in part, at least, to a general hostility to the changes on Starovoitov's part. Some of that is understandable; it can be frustrating to create a mechanism to solve a specific problem, only to be told that it needs to be generalized so that it is better suited to unrelated problems as well. But the kernel exists to address the entire community's problems, so this process of making features more generally useful is a vital part of the kernel's long-term success. At least some of the points raised in the discussion will need to be addressed before ELF modules can find their way into the mainline kernel.
