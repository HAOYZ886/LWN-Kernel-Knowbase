---
title: Special file descriptors in BPF
url: https://lwn.net/Articles/933182/
date: "June 5, 2023"
category: BPF
author: "By Jonathan Corbet June 5, 2023"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 5, 2023

Developers learning the Unix (or POSIX in general) system-call set will quickly encounter file descriptors, which are used to represent open files and more. Developers also tend to learn early on that the first three file descriptors are special, with file descriptor zero being the standard input stream, one being standard output, and two being standard error. The kernel, though, does not normally attach any specific meaning to a given descriptor number, so it was somewhat surprising when a recent BPF patch series attempted to attach a special meaning to zero when used as a file descriptor. 

BPF objects (maps and such) normally go away when they are closed, usually when the creating process exits. They may be "pinned", though, which gives them a name in the BPF filesystem (usually under `/sys/fs/bpf`) and allows them to outlive the creating process. The existing API for the pinning of BPF objects is path-based, meaning that the caller provides a string containing the full path name to be created for an object. 

The [series in question](<https://lwn.net/ml/bpf/20230516001348.286414-1-andrii@kernel.org/>), posted by Andrii Nakryiko, adds a mechanism to pin BPF objects using an open file descriptor instead of a path name to identify the containing directory. The feature itself is widely seen as a useful addition and is not controversial, but one aspect of it was. The patch series expanded an anonymous structure in the impressively sprawling [`bpf_attr` union](<https://elixir.bootlin.com/linux/v6.4-rc1/source/include/uapi/linux/bpf.h#L1319>) to contain a new `path_fd` field that would be used to provide the file descriptor identifying the directory in which (or below which) to pin the object. If `path_fd` were provided as zero, though, it would be interpreted to mean the calling process's current working directory — the same as the `AT_FDCWD` argument to system calls like [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>). 

The motivation behind these semantics is not that hard to understand. This new field will have a value of zero by default if the calling process does not provide it. Interpreting that zero as "start from the current working directory" allows for an easy, backward compatible expansion of the interface, since programs that do not provide that file descriptor will behave as they did before. But this interpretation also means that no process can use file descriptor zero as the starting directory for this system call, and that was seen as a source of concern. 

Christian Brauner quickly [questioned this decision](<https://lwn.net/ml/bpf/20230516-briefe-blutzellen-0432957bdd15@brauner/>), calling it ""very counterintuitive to userspace and pretty much guaranteed to cause subtle bugs"". He also asked if similar practices were to be found in other parts of the BPF subsystem. Nakryiko [answered in the affirmative](<https://lwn.net/ml/bpf/CAEf4BzafCCeRm9M8pPzpwexadKy5OAEmrYcnVpKmqNJ2tnSVuw@mail.gmail.com/>): 

> Yes, it's a very bpf()-specific convention we've settled on a while ago. It allows a cleaner and simpler backwards compatibility story without having to introduce new flags every single time. Most of BPF UAPI by now dictates that (otherwise valid) FD 0 can't be used to pass it to bpf() syscall. 

He added that the libbpf library goes out of its way to hide this behavior by reopening file descriptors that would otherwise be returned as zero. 

Brauner [did not like](<https://lwn.net/ml/linux-fsdevel/20230517-allabendlich-umgekehrt-8cc81f8313ac@brauner/>) this convention: 

> I personally find this extremely weird to treat fd 0 as anything other than a random fd number as it goes against any userspace assumptions and drastically deviates from basically every file descriptor interface we have. I mean, you're not just saying fd 0 is invalid you're even saying it means AT_FDCWD. 

There was [evidently](<https://lwn.net/ml/linux-fsdevel/CAADnVQKM3Jh7Sj7o9pz79Dme=kjinjsDPipE6MBUY-f=UfA+Tw@mail.gmail.com/>) some sort of off-list conversation, and Nakryiko later posted [an updated patch set](<https://lwn.net/ml/linux-fsdevel/20230518215444.1418789-1-andrii@kernel.org/>) that introduces a new flag to indicate whether a file descriptor has been provided rather than checking for a zero value. But it seems clear that the BPF community is not entirely thrilled by this change, and wants to continue to treat that value as special. Alexei Starovoitov [said](<https://lwn.net/ml/linux-fsdevel/CAADnVQLitLUc1SozzKjBgq6HGTchE1cO+e4j8eDgtE0zFn5VEw@mail.gmail.com/>) that the design of Unix includes two fundamental mistakes: allowing zero to be a valid file-descriptor value, and assigning special meanings to the first three descriptor values — a decision that he called ""just awful"". Those file descriptors, he said, are ""taken"" and should not be used for any other purpose; the BPF developers decided to implement part of that approach themselves: 

> Because of that, several years ago, we've decided to fix unix mistake #1 when it comes to bpf objects and started reserving fd=0 as invalid. This patch is proposing to do the same for path_fd (normal vfs fd) when it is passed to bpf syscall. I think it's a good trade-off and fits the rest of bpf uapi. 

Brauner [reiterated](<https://lwn.net/ml/linux-fsdevel/20230518-erdkugel-komprimieren-16548ca2a39c@brauner/>) that disallowing zero as a file descriptor seemed strange, but clarified his bigger complaint with the first version of the patch set: it didn't just disallow that file descriptor, but assigned a separate and incompatible meaning to it. If every kernel subsystem could attribute its own meaning to specific file-descriptor numbers, he said, the results would quickly spiral out of control. 

Ted Ts'o [wondered](<https://lwn.net/ml/linux-fsdevel/ZGY2eICf8Ndr3Xg5@mit.edu/>) if there could be security problems inherent in the BPF approach and suggested that it should perhaps be changed even if ABI incompatibility resulted. Nobody else, though, has raised security concerns with regard to this practice. 

Linus Torvalds was [highly critical](<https://lwn.net/ml/linux-fsdevel/CAHk-=wgmRTogGmR8E_SYOiHFpz8cY+0xj7nBpv9UwGU6k-UPAA@mail.gmail.com/>) of any code that treats specific file descriptors specially: 

> But fd 0 is in absolutely no way special. Anything that thinks that a zero fd is invalid or in any way different from (say) fd 5 is completely and utterly buggy by definition. [...] 
> 
> If bpf thinks that 0 is not a file descriptor, then bpf is simply wrong. No ifs, buts or maybes about it. It's like saying "1 is not a number". It's nonsensical garbage. 

Brauner [acknowledged](<https://lwn.net/ml/linux-fsdevel/20230518-gebrechen-tulpen-7be50e0f5b1a@brauner/>), though, that is is probably too late to change the BPF subsystem's practice of treating file descriptor zero as invalid. That policy has been baked into the code for years, so it would be surprising if no code in the wild depended on it. The patch assigning a more special meaning to that descriptor number was intercepted in time, though, and it doesn't appear that this practice exists anywhere else in the code. So programs interacting with BPF will be unable to use file descriptor zero as an ordinary descriptor, but neither will they be expected to use it as a special value.
