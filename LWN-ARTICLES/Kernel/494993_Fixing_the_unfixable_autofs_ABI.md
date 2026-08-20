---
title: Fixing the unfixable autofs ABI
url: https://lwn.net/Articles/494993/
date: "April 30, 2012"
category: "Development model-User-space ABI; Filesystems-autofs"
author: "By Jonathan Corbet April 30, 2012"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 30, 2012

One of the few hard rules of kernel development is that breaking the user-space binary interface is not acceptable. If there is user-space code that depends on specific behavior, that behavior must be maintained regardless of how inconvenient that may be. But what is to be done if two different programs depend on mutually-incompatible behaviors, so that it is seemingly impossible to keep them both working? The answer may be to violate another rule by putting an ugly hack into the kernel—or to do something rather more tricky. 

The "autofs" protocol is used to communicate between the kernel and an automounter daemon. It allows the automounter to set up special virtual filesystems that, when referenced by user space, can be replaced by a remote-mounted real filesystem. Much of this protocol is implemented with `ioctl()` calls on a special autofs device, but it also makes use of pipes between the kernel and user space when specific filesystems are mounted. 

This protocol is certainly part of the kernel ABI, so its components have been defined with some care. One of the key elements of the autofs protocol is the `autofs_v5_packet` structure, which is sent from the kernel to user space via a pipe; it is used, among other things, to report that a filesystem has been idle for some time and should be unmounted. This structure looks like: 

```
struct autofs_v5_packet {
    	struct autofs_packet_hdr hdr;
    	autofs_wqt_t wait_queue_token;
    	__u32 dev;
    	__u64 ino;
    	__u32 uid;
    	__u32 gid;
    	__u32 pid;
    	__u32 tgid;
    	__u32 len;
    	char name[NAME_MAX+1];
        };
```

The size of every field is precisely defined, so this structure should look the same on both 32- and 64-bit systems. And it does, except for one tiny little problem. The size of the structure as defined is 300 bytes, which is not divisible by eight. So if two of these structures were to be placed contiguously in memory, the 64-bit `ino` field would have to be misaligned in one of them. To avoid this problem, the compiler will, on 64-bit systems, round the size of the structure up to a multiple of eight, adding four bytes of padding at the end. So `sizeof()` on `struct autofs_v5_packet` will return 300 on a 32-bit system, and 304 on a 64-bit system. 

That disparity is not a problem most of the time, but there is an exception. Automounting is one of the many tasks being assimilated by the systemd daemon. When systemd reads one of the above structures from the kernel, it checks the size of what it read against its idea of the size of the structure to ensure that everything is operating as it should be. That check works just fine, as long as systemd and the kernel agree on that size. And normally they do, but there is an exception: if systemd is running as a 32-bit process on a 64-bit kernel, it will get a 304-byte structure when it is expecting 300 bytes. At that point, systemd concludes that something has gone wrong and gives up. 

In February, Ian Kent merged [a patch](<http://git.kernel.org/linus/a32744d4abae24572eff7269bc17895c41bd0085>) to deal with this problem. One could be forgiven for calling the solution hacky: on 64-bit systems, the kernel's automount code will subtract four from the size of that structure if (and only if) it is talking with a user-space client running in 32-bit mode. This patch makes systemd work in this situation; it was merged for 3.3-rc5 and fast-tracked into the various stable kernel releases. Everybody then lived happily ever after. 

...except they didn't. It seems that the `automount` program from the autofs-tools package, which is still in use on a great many systems, had run into this problem a number of years ago. At that time, the autofs-tools developers decided to work around the problem in user space. So, if automount determines that it is running in 32-bit mode on a 64-bit kernel (Linus [has little respect](<https://lwn.net/Articles/495003/>) for how that determination is done, incidentally), it will correct its idea of what the structure size should be. If the kernel messes with that size, the automount "fix" no longer works, so Ian's patch fixes systemd at the cost of breaking automount. 

So we are now in a situation where two deployed programs have different ideas of how the autofs protocol should work. On pure 32- or 64-bit systems, both programs work just fine, but, depending on which kernel is being run, one or the other of the two will break in the 32-on-64 configuration. If Ian's patch remains, some users will be most unhappy, but reverting it will upset other users. It is, in other words, a somewhat unfortunate situation. 

Unfortunate, but not necessarily unrecoverable. One possible way to fix things can be seen in [this patch](<https://lwn.net/Articles/495004/>) from Michael Tokarev. In short, this patch looks at the name of the current command (`current->comm`) and compares it against "`automount`". If the currently-running program is called "automount," the structure-size tweak is not applied and things work again. For any other program (including systemd), the previous fix remains. So things are fixed at the expense of having the kernel ABI depend on the name of the running program. At best, this solution can be described as "inelegant." At worst, there may be some other, unknown program with a different name that breaks in the same way automount does; any such program will remain broken with this fix in place. 

Still, Linus has [conceded](<https://lwn.net/Articles/495008/>) that ""it's probably what we have to go with"". But he preferred to look for a less kludgy and more robust solution. One possibility was for the kernel to look at the size of the `read()` operation that would obtain the `autofs_v5_packet` structure prior to writing that structure; if that size is either 300 or 304, the kernel could give the calling program the size it is expecting. The problem here is that the `read()` operation is hidden behind the pipe, so the autofs code does not actually have access to the size of the buffer provided by user space. 

So Linus came up with a different solution, the concept of "[packetized pipes](<https://lwn.net/Articles/495081/>)". A packetized pipe resembles the normal kind with a couple of exceptions: each `write()` is kept in a separate buffer, and a `read()` consumes an entire buffer, even if the size of the read is smaller than the amount of data in the buffer. With a packetized pipe, the kernel can always just write the larger (304-byte) structure size; if user space is only trying to read 300 bytes, then it will get what it expects and be happy. So there is no need for special hacks in the kernel, just a slightly different type of pipe dynamics. Following a suggestion from Alan Cox, Linus made an open with `O_DIRECT` turn on the packetized behavior, so user space can create such pipes if need be. 

After a couple of false starts, Linus got this patch working and merged it just prior to the 3.4-rc5 release. So the 3.4 kernel should work fine for either automount or systemd. 

The kernel community got a bit lucky here; it was possible for a suitably clever and motivated developer to figure out a way to give both programs what they expect and make the system work for everybody. The next time this kind of problem arises, the solution may not be so simple. Maintaining ABI stability is not always easy or fun, but it is necessary to keep the system viable in the long term.
