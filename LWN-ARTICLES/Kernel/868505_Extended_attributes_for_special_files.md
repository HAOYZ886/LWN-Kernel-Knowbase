---
title: Extended attributes for special files
url: https://lwn.net/Articles/868505/
date: "September 9, 2021"
category: "Filesystems-Extended attributes; Virtualization"
author: "By Jonathan Corbet September 9, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 9, 2021

The Linux extended-attribute mechanism allows the attachment of metadata to files within a filesystem. It tends to be little used — at least, in the absence of a security module like SELinux. There is interest in how these attributes work, though, as evidenced by the discussions that have followed the posting of revisions of [this patch by Vivek Goyal](<https://lwn.net/ml/linux-kernel/20210902152228.665959-1-vgoyal@redhat.com/>), which seeks to make a seemingly small change to the rules regarding extended attributes and special files. 

Specifically, extended attributes (often referred to as "xattrs") are name-value pairs that can be attached to a file. The name of an extended attribute is divided into a namespace and an identifier within the namespace. The namespace is currently one of `security`, `system`, `user`, or `trusted`; each namespace has its own special rules. As a general rule, `system` and `trusted` see little use. The `security` namespace, instead, is used by a number of Linux security modules. An example of a `security` attribute can be seen by running `getfattr` on a system that is set up for SELinux: 

```
cd /; getfattr -d -m - .
        # file: .
        security.selinux="system_u:object_r:root_t:s0"
```

This is the SELinux label, which is used to determine which access policies apply to this directory. A process must have the `CAP_SYS_ADMIN` capability to change attributes in the `security` namespace; even then, a running security module might have its own ideas regarding whether to allow a change to happen. 

The `user` namespace is entirely unprivileged; users may assign whatever attributes they like to any files they are capable of writing. There is an exception, though: it is not possible to attach `user` extended attributes to symbolic links or device special files. The [man page](<https://man7.org/linux/man-pages/man7/xattr.7.html>) describes this restriction as a control on resource consumption; if any process that can write to `/dev/null` can attach extended attributes to it, the disk could quickly fill with attribute data. It is still possible for a suitably privileged process to attach `security` extended attributes to those files. 

Goyal's patch relaxes that restriction slightly, so that a process that owns a symbolic link or device special file would be able to attach `user` extended attributes to that file. That allows users to change files they have control over anyway while avoiding the `/dev/null` problem. It seems to many like a reasonable change to make, but Andreas Gruenbacher [opposed it](<https://lwn.net/ml/linux-kernel/CAHc6FU4foW+9ZwTRis3DXSJSMAvdb4jXcq7EFFArYgX7FQ1QYg@mail.gmail.com/>): 

> The idea behind user.* xattrs is that they behave similar to file contents as far as permissions go. It follows from that that symlinks and special files cannot have user.* xattrs. This has been the model for many years now and applications may be expecting these semantics, so we cannot simply change the behavior. 

The possibility of breaking user space in subtle ways is indeed worrisome; Goyal has [suggested](<https://lwn.net/ml/linux-kernel/YTEcYkAA2F1FhOvF@redhat.com/>) that the new behavior could be made opt-in to avoid creating such surprises. But the bigger sticking point has to do with the intended use case for this feature. Kernel developers will normally ask why a new behavior is needed; that information is necessary to evaluate whether the proposed solution is indeed the best way to solve the problem. In this case, not all of them were happy with the answer. 

Goyal is working on virtualization and, in particular, with use cases where guests want to share a filesystem with the host. The kernel's [virtiofs filesystem](<https://www.kernel.org/doc/html/latest/filesystems/virtiofs.html>) is designed for this application; it functions as a sort of FUSE filesystem that runs over Virtio, so it performs reasonably well. On the host side, the [virtiofsd daemon](<https://qemu.readthedocs.io/en/latest/tools/virtiofsd.html>) performs the actual filesystem access needed by the client. 

Virtiofsd tries to drop as many capabilities as possible, though it must still be able to manipulate files owned by any user and group ID. It seems to work well, except for one little problem: `security` extended attributes. The client may want to manipulate those attributes within the filesystem, but the host may prevent virtiofsd from carrying out those changes. In particular, if the host is running a security module like SELinux, that module will be placing its own interpretation on those attributes and may not be appreciative of the changes that virtiofsd is trying to make. Changing those extended attributes requires `CAP_SYS_ADMIN` in any case, and virtiofsd is trying to run without that capability. 

The answer to the problem, naturally, is another layer of indirection. When properly configured, virtiofsd is able to remap extended attributes between the guest and the host; it can, for example, be made to turn the `security` attributes managed by the guest into `user` attributes on the host. No privilege is required for virtiofsd to change `user` attributes, so everything works as desired. 

More accurately, _almost_ everything works as desired. As Goyal notes, when the guest goes into the dreaded relabeling coma that SELinux seems to require from time to time, errors result. The problem is the restriction on `user` extended attributes for special files; the guest is trying to relabel its devices, but the host cannot assign those labels (remapped as `user` extended attributes) to the appropriate special files. That problem is what is driving Goyal's patch relaxing the restrictions, which makes this use case work. 

Casey Schaufler [let it be known](<https://lwn.net/ml/linux-kernel/79dcd300-a441-cdba-e523-324733f892ca@schaufler-ca.com/>) that he had no problem with assigning `user` extended attributes to special files. But the remapping done in virtiofsd, he said, is ""unreasonable"": 

> As I have stated before, this introduces a breach in security. It allows an unprivileged process on the host to manipulate the security state of the guest. This is horribly wrong. It is not sufficient to claim that the breach requires misconfiguration to exploit. Don't do this. 

Schaufler and Goyal have gone around on this topic numerous times over the three revisions of this patch set. Schaufler contends that the only secure way to handle a situation like this is for host-side changes to require the same level of privilege as is required on the guest. So, for example, it would be acceptable to remap the `security` extended attributes into the `trusted` namespace (which also requires `CAP_SYS_ADMIN`) instead. Goyal, meanwhile, continues to look for a solution that does not require additional privilege in virtiofsd. 

That solution, in the end, may take a bit of a different form, inspired by [this observation](<https://lwn.net/ml/linux-kernel/YTYoEDT+YOtCHXW0@work-vm/>) from David Alan Gilbert: 

> IMHO the real problem here is that the user/trusted/system/security 'namespaces' are arbitrary hacks rather than a proper namespacing mechanism that allows you to create new (nested) namespaces and associate permissions with each one. 
> 
> Each one carries with it some arbitrary baggage (trusted not working on NFS, user. having the special rules on symlinks etc). 

Miklos Szeredi [suggested](<https://lwn.net/ml/linux-kernel/CAJfpegvbkmdneMxMjYMuNM4+RmWT8S7gaTiDzaq+TCzb0UrQrw@mail.gmail.com/>) a next step (evidently originally proposed by Eric Biederman) to adapt `trusted` extended attributes for use within [user namespaces](<https://lwn.net/Articles/532593/>). If a given user namespace is owned by user ID 1000, then `trusted.foo` within the namespace would be stored (and visible) in the initial namespace as something like `trusted<1000>.foo`. Virtiofsd could be run within a user namespace and use the `trusted` extended attributes without any extra privilege, and security modules on the host would still have a say in how those attributes are managed. Goyal [agreed](<https://lwn.net/ml/linux-kernel/YTfcT1JUactPhwSA@redhat.com/>) that this approach might work. 

The only problem is that somebody has to implement the new behavior for user namespaces — a complex part of the system where it is easy to make security-wrecking mistakes. Biederman has already [pointed out](<https://lwn.net/ml/linux-kernel/87lf47tahb.fsf@disp2133/>) one potential problem related to nested namespaces. But, assuming that feature can be safely provided, Goyal's original problem should be solvable. _This_ is why kernel developers tend to be inquisitive about the use cases driving a proposed change: the best solution to a problem is often not the first one that comes to mind.
