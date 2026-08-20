---
title: Splicing out vmsplice()
url: https://lwn.net/Articles/1075838/
date: "June 4, 2026"
category: "System calls-vmsplice; splice"
author: "By Jonathan Corbet June 4, 2026"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 4, 2026

The [`splice()`](<https://man7.org/linux/man-pages/man2/splice.2.html>) and [`vmsplice()`](<https://man7.org/linux/man-pages/man2/vmsplice.2.html>) system calls are meant to improve performance for certain data-movement tasks by minimizing (or avoiding altogether) system calls and the copying of data. They also have a long history of security problems. The recent flood of LLM-discovered vulnerabilities has drawn attention, once again, to `splice()` and `vmsplice()`; as a result, they may end up being removed altogether. 

#### Some history

Larry McVoy is credited for first raising the idea of a `splice()` system call that would connect a file directly to a pipe. With the classic POSIX API, an application would copy file data into a pipe with a loop that read chunks of data from the file (thus copying that data into user space), then wrote those chunks to the pipe (copying the data back into the kernel). With a single `splice()` call, the application could request that the kernel implement that loop, getting the work done with far fewer system calls and less data copying. After years of discussion, a `splice()` implementation was [added in 2006](<https://lwn.net/Articles/178199/>) to the 2.6.17 kernel by Jens Axboe; it looks like: 

```
ssize_t splice(int fd_in, off_t *off_in, int fd_out, off_t *off_out,
        		   size_t size, unsigned int flags);
```

It will attempt to copy up to `size` bytes from `fd_in` to `fd_out`; one of the two file descriptors must be a pipe. The return value is the number of bytes actually copied. 

`vmsplice()` was [added (by Axboe) shortly thereafter](<https://lwn.net/Articles/181169/>) (also in time for 2.6.17): 

```
ssize_t vmsplice(int fd, const struct iovec *iov, size_t nr_segs, unsigned int flags);
```

Here, `iov` is an array of `nr_segs` [`iovec`](<https://elixir.bootlin.com/linux/v7.0.10/source/include/uapi/linux/uio.h#L17>) structures indicating regions of memory. If `fd` is a readable pipe file descriptor, data will be read into those memory regions from the pipe. If, instead, `fd` is writable, the data will move _from_ the memory regions into the pipe. The fact that there is no explicit argument indicating the direction of data movement is one of `vmsplice()`'s special quirks. Another is that there is no way to know when the data transfer completes and, thus, when it is safe to access the memory given to `vmsplice()`. The `SPLICE_F_GIFT` flag "gifts" the indicated memory pages to the kernel; the caller pledges to never touch them again. This option is meant to make zero-copy operations available in some situations. 

The implementation of the splice system calls involves a fair amount of complexity within the kernel; it also depends on all kernel subsystems that might receive a spliced buffer to handle it properly. So, arguably, it is not surprising that they have been the focus of a lot of vulnerabilities, including [a high-profile exploit](<https://lwn.net/Articles/268783/>) (see also [this followup article](<https://lwn.net/Articles/271688/>)) in 2008. Many of the recently disclosed kernel vulnerabilities involve a combination of these system calls and subsystems that do not handle them correctly. 

#### Protecting read-only files

In mid-May, Pedro Falcato sent [a brief patch](<https://lwn.net/ml/all/20260516182126.530498-1-pfalcato@suse.de>) aimed at making the splice system calls harder to exploit. Specifically, the patch adds a new sysctl knob, `fs.splice_needs_write`; if that knob is set to a value of one (the default is zero), then it will not be possible to `splice()` to a file that the calling process lacks the permissions to write to, even if the requested operation is a read from that file that would otherwise be permitted. Similarly, `vmsplice()` cannot be invoked with memory backed by an unwritable file. 

In essence, this patch is an admission of defeat; it is an acknowledgment that the splice system calls simply cannot be implemented in a way that prevents security vulnerabilities. Rather than (continue to) try, the kernel developers would simply be giving administrators the ability to forbid splice operations that might be exploited to give write access to a read-only file. If more such vulnerabilities exist, this change would be a quick way to render them all harmless. 

The reactions to the proposal were mixed. Matthew Wilcox [said](<https://lwn.net/ml/all/agj4mXKRVW44ZJ18@casper.infradead.org>): ""I don't have a problem with the idea, other than it's really sad we have to do this"". Christian Brauner, though, [called it](<https://lwn.net/ml/all/20260518-starten-messdaten-3b8aa670ec85@brauner>) ""a knee-jerk reaction to an exploit class originating in buggy modules that we have little control over"" and an extension of an already problematic API. Jann Horn [suggested](<https://lwn.net/ml/all/CAG48ez0jbSUgT3ZxPKZP7Eu=K7ce2cX7k2NzHCHNMOxQjOGT9w@mail.gmail.com>) that, rather than blocking operations on read-only files, it would be better to degrade the call to an ordinary copy operation. Mateusz Guzik [called it](<https://lwn.net/ml/all/CAGudoHHeYbPWQbz+vXoS-Oi4PhxX6rh5XsMUkZetyfdnJHNj=g@mail.gmail.com>) ""a half-measure which will at best buy few weeks until splice bugs dry out and there will be a new attack vector du jour which people point their LLMs at"". 

After the discussion had gone on for a few days, Falcato [said](<https://lwn.net/ml/all/ahg6JgO0wUkJKjRb@pedro-suse>) that the consensus seemed to favor degrading to simple copy operations rather than blocking the system call entirely. There would be a second version of the series forthcoming that took that approach. 

#### Removing `vmsplice()`

Before that second version could appear, though, Askar Safin showed up with [a patch series](<https://lwn.net/ml/all/20260531010107.1953702-1-safinaskar@gmail.com>) that takes away the special functionality of `vmsplice()` entirely. The system call still exists, but the implementation simply copies the data within the kernel rather than attempting to provide complex, zero-copy semantics. In short, a `vmsplice()` call would be turned into the equivalent [`preadv2()` or `pwritev2()`](<https://man7.org/linux/man-pages/man2/readv.2.html>) call. 

Falcato [was unimpressed](<https://lwn.net/ml/all/ahv16ogY8Zx3Rtox@pedro-suse.lan>) with this development, and suggested that Safin's patches should not even be considered. Brauner had [some gentle criticism](<https://lwn.net/ml/all/20260601-geldentwertung-aufdecken-aussehen-1502bfad440d@brauner>) for the way in which this work was done: 

> So I think this is a case where no explicit rules have been broken. But if you know that someone has been posting patches and is working on a problem just racing them to get your own stuff merged is very likely to unnecessarily ruffle feathers. So sync with the person next time. 

The patches themselves, though, have been reasonably well received. Andy Lutomirski [said](<https://lwn.net/ml/all/CALCETrW__=8mSusayfXG7UFCfue5BGbx+vqESj1d9wqOfX4s8w@mail.gmail.com>): 

> I have no comment on the code or the history. But I'm 100% in favor of the solution. vmsplice is a crappy API, and would be incredibly complex to get the implementation right, and it should be removed. But it has users, and the approach of just mapping them straight to pread/pwrite makes perfect sense. 

Linus Torvalds was [cautiously in favor](<https://lwn.net/ml/all/CAHk-=wiFuud0Nn3B9YpTWyQja08TeXVk2AB-aAkmVXyigOagbQ@mail.gmail.com>) of the change; he also [suggested](<https://lwn.net/ml/all/CAHk-=wifX_rrDjRGnDnOqE-usptAukuXKrmuPuVDP5bOCBWzGQ@mail.gmail.com>) making a similar change to `splice()` if the `vmsplice()` change does not cause too much anguish. Brauner, for his part, has [applied the series](<https://lwn.net/ml/all/20260601-enthusiasmus-canceln-anlehnen-0e62317a9784@brauner>) with an eye toward merging during the 7.2 development cycle. 

That merging should not be seen as a certainty at this point; it is noteworthy that this conversation has happened mostly without the participation of developers who actually _use_ the splice calls. Some of those users are beginning to appear now. Christian Brauner [passed on](<https://lwn.net/ml/all/20260603-raumfahrt-unmerklich-ertrugen-c4ecae70d5f9@brauner>) a report of a test regression pointing out a subtle behavior change that can probably be addressed. Willy Tarreau [said](<https://lwn.net/ml/all/aiEb8CTM-ovMIq7-@1wt.eu>) that he is a heavy user of the splice system calls: ""It simply doubles the network bandwidth compared to not using that. (62 Gbps per core vs 31). I would seriously miss it if I couldn't use this anymore."" He suggested perhaps further restricting the types of memory that could be passed to `vmsplice()` (such as only allowing anonymous memory) instead. 

So users of the splice system calls do exist, but there seem to be a lot of voices united in their desire to remove the zero-copy logic behind those calls. Torvalds has also [indicated](<https://lwn.net/ml/all/CAHk-=wiEwSjfbjfO74xu=UmkkdHXkJg5QNQ8pP-3iYmunmeV9g@mail.gmail.com>) a desire to make a similar change to the more widely used [`sendfile()`](<https://man7.org/linux/man-pages/man2/sendfile.2.html>) system call which, he said, was ""a mistake"". The reimplementation of these system calls should not break any code, since the resulting behavior should look the same from user space, but it does have the possibility of causing performance regressions. That may be enough to prevent these changes from happening in the end. But, as Torvalds [said](<https://lwn.net/ml/all/CAHk-=wizkDXRut5xLXRF-CVUVYMaZ5AOexxeghOAoXPb4yAvQg@mail.gmail.com>): ""I just suspect we'll never get real answers without going the 'let's just see what happens' route"". The time has apparently come to see what happens.
