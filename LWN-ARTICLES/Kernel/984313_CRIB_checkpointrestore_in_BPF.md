---
title: "CRIB: checkpoint/restore in BPF"
url: https://lwn.net/Articles/984313/
date: "August 7, 2024"
category: BPF; Checkpointing
author: "By Jonathan Corbet August 7, 2024"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 7, 2024

The desire for the ability to checkpoint a process — to record its state in a form that can be restarted at a future time — on Linux is almost as old as Linux itself. See, for example, [this announcement](</1998/0528/a/checkpoint.html>) of a checkpoint project that appeared in LWN in 1998. While working solutions exist, they can be somewhat fragile and difficult to use; it is not surprising that some people are interested in finding a better alternative. A current effort goes by the name CRIB, for Checkpoint/Restore in (naturally) BPF. It is far from clear that CRIB will replace the existing solutions, but it is an interesting look at a different way of solving the problem. 

A checkpoint/restore solution must overcome two challenges, neither of which is easy. On the checkpoint side, it is necessary to obtain a complete description of a process (or set of processes), with no important details overlooked; that requires collecting a lot of information that the kernel was not designed to export. On the restore side, that information must be used to recreate the checkpointed process(es), possibly on a different system, in such a way that those processes cannot tell the difference — once again, using interfaces that were not designed for this purpose. 

Early efforts to implement checkpoint/restore functionality focused on the kernel. There was a patch set that first [started getting serious attention](<https://lwn.net/Articles/293575/>) in 2008 that added two new system calls (`checkpoint()` and `restart()`) to do all of the work. The former would write the entire state of a process to a given file, while the latter would restore the process from a file. This work added quite a bit of complexity to the kernel and never really got to the point where it could reliably checkpoint and restore processes. Kernel developers were concerned about the challenges of maintaining a feature that was widely intrusive even in its incomplete state, and about whether it would ever reach the needed level of completeness and reliability. More than two years later, this work was [still being discussed](<https://lwn.net/Articles/414264/>), but there was a clear appetite for alternatives. 

#### Moving to user space

In 2011, Pavel Emelyanov showed up with the first [checkpoint/restore in user space (CRIU) patches](<https://lwn.net/Articles/452184/>) that moved that work out of the kernel. This approach, which had its origins in the [OpenVZ project](<https://wiki.openvz.org/Main_Page>), attracted a lot of interest, but it was still a long path toward a working solution. Kernel interfaces were not developed with the idea of providing enough information to completely describe a process, or to recreate a process that matches the original in every detail at a later time. It is worth thinking about all of the information that is needed, including: 

  * All of the threads running within the process, where they are executing, their priority, and their signal-handling state. 
  * A complete memory map of the process: which mappings exist at which addresses and the protections that apply to each. 
  * A list of the process's open files, including the actual files that have been opened, whether each was opened for read, write, or append access, the current file position, and the file-descriptor number. 
  * Every open network connection, who the peer is, which protocol is in use, and any in-transit data. 
  * The configuration of the namespaces in which the process is running. 
  * Active file notifications, terminal configurations, active timers, and no end of other details. 

On the other side, any solution must be able to restore all of this data, creating a running process with no surprising changes in its configuration or environment. 

Much of this information was already available, perhaps inefficiently, from the kernel via the system-call interface and `/proc`. But, in many cases, additional support was needed to get to a working solution; the developers behind the effort that eventually turned into [CRIU](<https://criu.org/Main_Page>) spent years on that project. They added features like [TCP connection hijacking](<https://lwn.net/Articles/454427/>) and [connection repair](<https://lwn.net/Articles/495304/>), more information in `/proc`, the [`kcmp()` system call](<https://lwn.net/Articles/478111/>), [time namespaces](<https://lwn.net/Articles/766089/>) (so that a sudden time jump does not cause a restarted process to misbehave), and many others over time. In 2024, CRIU is, as a result of this long effort, a working solution with an active user and developer base. 

CRIU is not without its shortcomings, though, many of which derive from the fact that CRIU must rely on a wide range of kernel interfaces that, for the most part, were not designed to support the checkpoint and restore operations. Checkpointing a process requires opening a large set of files in `/proc` and `/sys`, which slows things down considerably, and each of those files has its own special format that must be parsed. CRIU is easily broken as the kernel evolves. Newer features, such as io_uring, are difficult to support — if they can be supported at all. Similar challenges exist on the restore side of the problem. 

#### Moving back into the kernel with BPF

Juntong Deng thinks that the solution to these problems lies in BPF; the [current CRIB patches](<https://lwn.net/ml/all/AM6PR03MB58488045E4D0FA6AEDC8BDE099A52@AM6PR03MB5848.eurprd03.prod.outlook.com>) have been posted as a proof of the concept. The core idea is that a user-space checkpoint application would load a BPF program to obtain the necessary information directly from the kernel. It would marshal and format that data, then feed it to user space by way of a fast ring-buffer interface. The user-space piece would end up with an interface that provides functionality similar to the `checkpoint()` system call that was proposed years ago, without the complexity that CRIU must manage. 

Of course, that complexity does not go away entirely; it is instead pushed into the BPF part of the system. A suitably privileged BPF program has read access to much of the kernel, so it could obtain a great deal of the needed information without any special support. It can simply read the kernel data structures directly. For the more complex cases where digging through kernel structures is not practical, special-purpose kfuncs can be provided for the BPF program to use. For example, [this patch](<https://lwn.net/ml/all/AM6PR03MB5848DCE9DC6D96454E9C3EBD99A52@AM6PR03MB5848.eurprd03.prod.outlook.com>) adds a kfunc called `bpf_file_from_task_fd()`, which will return the `struct file` pointer corresponding to a process's file descriptor. The function will also take a reference to that file so that it will not vanish while the BPF program is reading it. Many of the other added kfuncs are focused on obtaining data from network sockets, which tend to have a complex internal state. 

Any BPF program used to checkpoint a process in this way is going to be strongly tied to a specific version of the kernel. In theory, BPF interfaces are not a part of the kernel's stable ABI, so the prospect of breaking checkpoint programs should not hinder ongoing kernel development. Kernel changes can break CRIU as well, of course, but CRIU depends on user-space interfaces that are not supposed to break; that suggests that a BPF-based checkpoint function might require more maintenance to keep up with the kernel. In exchange, though, it would have deeper and better access to the state of the processes it is checkpointing and should be quite a bit faster. 

The restore side of the problem might prove to be a bit more difficult. While a BPF program can be given the ability to freely read data from the kernel's address space, the same is not true for writing data. There is a set of macros that goes by the name of `BPF_CORE_READ()` that BPF programs use to read data. Deng, in the patch cover letter, suggested the addition of an equivalent `BPF_CORE_WRITE()` set, saying: 

> I am not sure what the current attitude of the kernel community towards BPF_CORE_WRITE is, personally I think it is well worth adding, as we need a portable way to change the value in the kernel. 

Followers of BPF development will not have been surprised when Alexei Starovoitov [made the kernel community's position clear](<https://lwn.net/ml/all/etzm4h5qm2jhgi6d4pevooy2sebrvgb3lsa67ym4x7zbh5bgnj@feoli4hj22so>): 

> I'm afraid BPF_CORE_WRITE cannot be introduced without breaking all safety nets. It will make bpf just as unsafe as any kernel module if bpf progs can start writing into arbitrary kernel data structures. So it's a show stopper. 

Deng [responded](<https://lwn.net/ml/all/AM6PR03MB5848CA34B5B68C90F210285E99B12@AM6PR03MB5848.eurprd03.prod.outlook.com>) that, without the ability to write arbitrary kernel data, the restore functionality cannot be easily implemented, so development on CRIB would focus on the checkpoint side for now. Kumar Kartikeya Dwivedi [disliked](<https://lwn.net/ml/all/CAP01T75na=fz7EhrP4Aw0WZ33R7jTbZ4BcmY56S1xTWczxHXWw@mail.gmail.com>) that idea, saying that it would be better to have the form of a restore solution in mind, even if it cannot be implemented immediately. 

Various other details of the series were discussed; they seem much more amenable to an agreed-upon solution. The ability to restore arbitrary kernel data structures would appear to be a real sticking point, though; it is not clear which direction an acceptable solution will take. The restore process could be carried out mostly in user space, as is done with CRIU now, or another set of kfuncs could be added to facilitate the restoration of a process's state. While this work has been [scheduled for discussion](<https://lpc.events/event/18/contributions/1812/>) at the upcoming Linux Plumbers Conference, the amount of progress that can be made in a 15-minute slot will be limited. The best guide to where this project is headed will be found in future postings of the patch series.
