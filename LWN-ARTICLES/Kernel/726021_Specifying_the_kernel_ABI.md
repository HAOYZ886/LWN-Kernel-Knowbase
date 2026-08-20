---
title: Specifying the kernel ABI
url: https://lwn.net/Articles/726021/
date: "June 21, 2017"
category: "Development model-User-space ABI"
author: "By Jake Edge June 21, 2017 OSS Japan"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
June 21, 2017

* * *

[OSS Japan](<https://lwn.net/Archives/ConferenceByYear/#2017-Open_Source_Summit_Japan>)

At [Open Source Summit Japan](<http://events.linuxfoundation.org/events/open-source-summit-japan>) (OSSJ)—OSS is the new name for LinuxCon, ContainerCon, and CloudOpen—Sasha Levin gave a talk on the kernel's application binary interface (ABI). There is an effort to create a kernel ABI specification that has its genesis [in a discussion about fuzzers](<https://lwn.net/Articles/705937/#abi>) at the 2016 Linux Plumbers Conference. Since that time, some progress on it has been made, so Levin described what the ABI is and the benefits that would come from having a specification. He also covered what has been done so far—and the the extensive work remaining to be done. 

[ ![\[Sasha Levin\]](https://static.lwn.net/images/2017/ossj-levin-sm.jpg) ](<https://lwn.net/Articles/726121/>)

An ABI is how one piece of compiled code talks to another, he said. For the kernel, the ABI is how user space communicates with the kernel. That takes the form of system calls, `ioctl()` commands, and, to some extent, the calls in the [virtual dynamic shared object](<https://lwn.net/Articles/615809/>) (vDSO). 

The current process for Linux development says that kernel patches cannot break programs that rely on the ABI. That means a program that runs on the 4.0 kernel should be able to run on the 5.0 kernel, Levin said. While breakage is not intended, there is nothing that would automatically prevent it from happening. There are no tools that would detect an ABI break, so the responsibility to find these kinds of problems falls to user space. If a user-space program breaks because of an ABI change, users and developers of that program can complain to the kernel developers and, hopefully, get the problem fixed upstream. 

The kernel ABI gets extended by some developer coming up with a new feature that has a user-space interface and, with luck, some documentation that describes it. Usually the maintainer will require documentation before merging the feature, he said. Normally, though, someone else will write the corresponding user-space code to use the new feature; that might be developers for the GNU C library (glibc) or QEMU, for example. Then the kernel and user-space developers test to find things that are broken on both sides of the interface; the code is "massaged until stuff works" between the two. 

But that process has a number of flaws. Basic validity checks are forgotten and can often reappear as security vulnerabilities later on. Effectively, these missing checks allow user space to cause the kernel to do things that were never planned for, which is always dangerous. There are also undefined behaviors on both sides of the interface because there is no complete specification. Even if all of the checks are made, there is still room for the kernel to end up performing operations that were not planned. The lack of a specification can also lead to problems down the road; failing to verify flags and other parameters mean that changes in the kernel can cause existing programs to break. 

Backward compatibility is supposed to be verified by the [Linux Test Project](<https://linux-test-project.github.io/>) (LTP) and other tools, which help, but the real verification is done by all of the user-space applications. LTP will catch big things, Levin said, but not the majority of backward compatibility breaks. 

The user-space ABI is broken "every other release or so"; usually they are small things that no one cares about, he said. But, one of the problems is that it can take some time before the new release gets widespread testing. In the meantime, some user-space program could start relying on the new ABI, while another program still relies on the unbroken, old behavior; that would be a recipe for a difficult-to-solve problem. 

Much of the validation that is done in the kernel is done on an ad hoc basis. There is no clear definition of what should be checked or how the checks should be done. Each system call typically has its own way of checking, which opens up room for bugs, he said. If there are 20 different versions of ways to check some kind of parameter and one of them gets fixed, it is common for the other 19 to be missed. He suggested that using "`git blame`" on any major system call will show missed checks; "look and you will be unpleasantly surprised". 

Even in user space there are lots of different implementations for system call parameter handling. For example, `strace` has its own library describing all of the system calls, C libraries do their own parameter checking, and different fuzzers all have their own way of generating system call parameters. That is just more duplication, which leaves more room for mistakes, Levin said. Most of those implementations are written by developers who aren't necessarily familiar with the kernel side of the interface as well. "It's a mess." 

The existing documentation, in the form of `man` pages, is "pretty good". But `man` pages only cover about 80% of the use cases; they are not supposed to completely document the ABI of the kernel. The documentation is meant to help programmers get their programs working with the kernel, thus it is a "summary briefing" rather than any kind of "contract". 

#### Contract

Today, the contract is embodied in the kernel code for a specific kernel version. The documentation is based on someone's understanding of the kernel code, which may be wrong, and the kernel code itself is subject to change. There generally is no proactive effort to see if an ABI change affects a particular user-space program; its users find out later when things break. 

Having a contract would kill multiple birds with one stone, he said. It would force the kernel and user space to behave in a specific way. The backward compatibility problems would disappear, since changes that affect it could be detected. It would prevent a whole class of errors between user space and the kernel. It would fully document the ABI and it would also allow code reuse for the ABI, with the usual benefits that brings. 

What would this contract look like? It should be human readable so that people can review it, but also should be machine readable so that it can be turned into code for tests or to use in the kernel and user space. Hopefully, it would only need to be written once but could be used by all of the potential consumers. As a starting point, the system call descriptions used by the [syzkaller fuzzer](<https://lwn.net/Articles/677764/>) look reasonable. They are used to create calls to system calls that are correct enough that they tickle new parts of the kernel code. 

On the kernel side, the contract would be used to generate code to verify parameters and return values as part of the ABI. The code would validate the input and output parameters based on the specification. It would try to prevent calls with invalid arguments from even getting to the real system call code. That would reduce the amount of validation checks needed in the individual system calls; for example, file descriptors could be verified in only one place and system calls could rely upon getting a valid one. 

For user space, it would make things easier for programs and libraries that access the kernel ABI. Instead of hoping the ABI is understood, user-space programs would have a guarantee of behavior instead. The contract would be made usable for projects like `strace` that already have to work with the ABI. Validation code based on the specification could be added to glibc and other C libraries as well. 

By generating the validation code and centralizing it, lots of code in both the kernel and user space will be removed. Fixes to the validation code will be shared on both sides of the ABI. In addition, backward compatibility problems will be detected more easily and quickly. It will be difficult to implicitly break the ABI. 

There are also benefits for users of the stable and long-term support (LTS) kernels. Right now, some are afraid to upgrade their kernels because they are worried about a new kernel breaking their user-space application. The contract would provide more assurance that those applications will still run correctly. Even though he is a maintainer of LTS kernels, he thinks they are a "pretty bad idea" overall; if you keep older kernels alive for many years, "things are bound to go wrong". He is hoping that an ABI contract would help to kill off LTS kernels to some extent by increasing the frequency that users are willing to do kernel upgrades. 

There are security benefits as well. Centralizing the code that is used by multiple user-space projects as well as the kernel will likely lead to more people scrutinizing that code. A fix that is found by one project will fix the others as well. Many of these kinds of bugs lead to CVE entries, so ultimately this could help reduce the number of vulnerabilities for Linux. 

There are also academic and other researchers interested in a specification of the kernel ABI. For example, safety researchers are particularly interested as some government agencies will not allow certain industries (nuclear power, for example) to run Linux because there is no specification to describe the limits of what the ABI provides. 

#### Plans

The "hard part" is what is being worked on now, Levin said, which is to determine the format for the specification. The `open()` and `close()` system calls are "pretty easy to describe", but other system calls are more complicated and have a lot of interactions with other system calls. There is a need to start documenting all of the system calls and `ioctl()` commands and to go beyond what is listed in the `man` pages. The `man` page may say that `EINVAL` is returned for a bad flag value, but the specification needs to say exactly what flag values will cause that return. That needs to be written by someone who is familiar with the system call, he said. 

Then those specifications need to be tested. That needs to be done without breaking existing user-space programs, but still providing user space a way to test the code. If the feature is governed by a kernel configuration option, most users won't change their distribution's setting, which may limit testing. There needs to be a way to do user-space testing that still allows existing applications to work while the specification is incomplete and changing. 

Levin was asked if he knew of other, similar efforts. He said that the Linux Foundation started a project and paid a company to create a specification of some kind back in 2008. That effort crashed when the economy tanked around the same time and he has never seen any of the results of that work. 

Another audience member asked about the performance of putting these checks into the system call path, which is considered to be a hot path in the kernel. Levin acknowledged that, but said the idea was to effectively move the checking out of the system call itself, so that would essentially shift the time spent. But there would be more checks and there would be some impact from jumping through another layer; he was hopeful that it would not be a blocker. 

In answer to another question, Levin said he has been doing some [preliminary work](<https://lwn.net/Articles/706606/>) with the syzkaller developers, but that there is a need for more developers. It is just now starting to get to a point where it is worth getting something into the kernel, he said, but there is much more work still to be done. 

[I would like to thank the Linux Foundation for travel assistance to Tokyo for Open Source Summit.]
