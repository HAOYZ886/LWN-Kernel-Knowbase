---
title: Private loop devices with loopfs
url: https://lwn.net/Articles/819625/
date: "May 7, 2020"
category: "Block layer-Loopback device; Loopback device"
author: "May 7, 2020 This article was contributed by Marta Rybczyńska"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

May 7, 2020

This article was contributed by Marta Rybczyńska

A loop device is a kernel abstraction that allows a file to be presented as if it were a physical block device. The typical use for a loop device is to mount a filesystem image stored in a file. Loop devices are global and shared between users, which causes a number of problems for container workloads where the instances are expected to be isolated from each other. Christian Brauner has been working on this problem; he has posted a [patch set](<https://lwn.net/ml/linux-kernel/20200424162052.441452-1-christian.brauner@ubuntu.com/>) solving it by adding a small virtual filesystem called loopfs.

Loop devices typically appear under `/dev` with names like `/dev/loop _N_`. The special `/dev/loop-control` file can be used to create and destroy loop devices or to find the first available loop device. Associating a file with a specific device, or setting other parameters like offsets or block sizes, is done with `ioctl()` calls on the device itself. The [`loop(4)` man page](<http://man7.org/linux/man-pages/man4/loop.4.html>) has the details on how it all works. 

Users generally need not deal with specific devices, though; they can be managed behind the scenes with a special [form of the `mount` command](<http://man7.org/linux/man-pages/man8/mount.8.html#LOOP-DEVICE_SUPPORT>):

```
mount /tmp/myimage.img /mnt/disk -o loop
```

This causes `mount` to locate an available loop device, associate it with `/tmp/myimage.img`, then mount that loop device onto `/mnt/disk`. Some administrators may prefer a different form of the same mount command that gives more control:

```
mount /tmp/myimage.img /mnt/disk -o loop=/dev/loop1
```

In this mode, the administrator specifies the exact loop device to use. An administrator who needs more control over loop devices may also use the [`losetup`](<http://man7.org/linux/man-pages/man8/losetup.8.html>) command to query and set up loop-device properties.

As noted above, loop devices are global and shared between users; `/dev/loop3` is the same device in all namespaces. If an application needs a private device, it has no way to request one. Loop devices are also, obviously, shared between containers, so one container can monitor the operations — or access the data — of the others.

A number of different use cases for loop devices were raised in the discussion of this patch set. Dmitry Vyukov [gave one example](<https://lwn.net/ml/linux-kernel/CACT4Y+aDeSAARG0b9FjDFyWuhjb=YVxpGtsvBmoKnHo+0TF4gA@mail.gmail.com/>): separating test processes from each other when they are using loop devices. He described the problems he has run into: 

Currently all loop devices and loop-control are global and cause test processes to collide, which in turn causes non-reproducible coverage and non-reproducible crashes. 

Brauner gave a number of examples from the container world. For example, [systemd-nspawn does not support loop devices](<https://lists.freedesktop.org/archives/systemd-devel/2017-August/039453.html>) as they cannot be discovered dynamically and owned by a container. Chromium OS [does not allow](<https://chromium.googlesource.com/chromiumos/docs/+/master/containers_and_vms.md#loop-mount>) the use of loop devices. Kubernetes has also [run into problems](<https://github.com/kubernetes-sigs/kind/issues/1333>) resulting from the global nature of loop devices: a file can remain bound to a device after its user has exited. 

#### loopfs

Loopfs is a new, in-kernel, virtual filesystem that implements the loop devices and the `loop-control` file. This filesystem can be mounted multiple times; the loop devices in each instance are independent from all other loop devices in all other instances. This allows private loop devices for applications and containers. Both the loop devices and the `loop-control` file in loopfs accept the same operations as the legacy ones.

One use of loopfs is to provide compatibility with old-style applications, but with virtualized loop-device files. In this case, the administrator can mount the filesystem and then replace the default loop control files with those from loopfs. Consider the following example, adapted from the patch cover letter:

```
# Mount a new loopfs instance in /dev/loopfs/
        mount -t loop loop /dev/loopfs/
    
        # Replace the standard loop control file with the ones from loopfs
        ln -sf /dev/loopfs/loop-control /dev/loop-control
    
        # Find the first available loop device
        loopdev=`losetup -f`     	  # will be something like /dev/loop0
        deventry=`basename $loopdev`  # now just "loop0"
    
        # Redirect that loop device to loopfs
        ln -sf /dev/loopfs/$deventry /dev/$deventry
    
        # mount an image
        mount -o loop /image.img /mnt/disk
```

There is a knob provided to control the maximum number of loop devices that can be created in any given loopfs instance; it can be found as `/proc/sys/user/max_loop_devices`. 

Christoph Hellwig [disagreed](<https://lwn.net/ml/linux-kernel/20200409075320.GA26234@infradead.org/>) with the loopfs approach, stating that the code is too big for the benefit it provides. Brauner [explained](<https://lwn.net/ml/linux-kernel/20200409083319.nlemf6d7g33hxhiy@wittgenstein/>) the additional use cases it allows, but the discussion stopped there. There have not been other substantial complaints about this proposal. 

Loopfs doesn't just allow an independent loop-device pool, it also opens a way to allow unprivileged users to mount loop devices. This can be enabled by combining loopfs with Brauner's earlier work on [system-call interception](<https://lwn.net/ml/linux-kernel/20190920083007.11475-1-christian.brauner%40ubuntu.com/>), which uses [seccomp](<https://lwn.net/Articles/656307/>) to establish a separate process to make decisions on which operations can be allowed. In such a setup, the unprivileged user can run `mount` as usual; the privileged process intercepting the system call will perform the actual operation.

Jann Horn [outlined](<https://lwn.net/ml/linux-kernel/CAG48ez0KWgLMOp1d3X1AcRNc4-eF1YiCw=PgWiGjtM6PqQqawg@mail.gmail.com/>) one possible problem with loop-device usage by unprivileged applications: most filesystem implementations are not prepared to deal with malicious filesystem images. While some work has been done, filesystem images are still generally treated as trusted data; that is why previous attempts to allow unprivileged filesystem mounting [have run into opposition](<https://lwn.net/Articles/755593/>) in the past. If an attacker has the ability to modify the image on the fly — as they would if they had access to the loop device providing that image — the problem would be compounded. 

Stéphane Graber [pointed out](<https://lwn.net/ml/linux-kernel/CA+enf=uhTi1yWtOe+iuv2FvdZzo69pwsP-NNU2775jN01aDcVQ@mail.gmail.com/>) that an implementation based on system-call interception does not have to mount filesystems directly; a FUSE-based mount could be used instead. That would prevent any filesystem-level vulnerabilities from turning into kernel vulnerabilities. The [LXD](<https://linuxcontainers.org/lxd/introduction/>) implementation allows both types of mount.

#### Next steps

Loopfs seems to solve a problem that users experience in practice. It has had three iterations in a week's time, addressing the comments given during the review. It may still take some time to find its way into the mainline kernel, but it is clear is that there numerous users waiting for a solution to the loop-device sharing issue.
