---
title: Improving fget() performance
url: https://lwn.net/Articles/787473/
date: "May 6, 2019"
category: "Memory management-Scalability"
author: "By Jake Edge May 6, 2019 LSFMM"
---

By **Jake Edge**  
May 6, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The performance of the `fget()` function in the kernel was the topic of a discussion led by Dave Watson at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). [`fget()`](<https://elixir.bootlin.com/linux/latest/source/fs/file.c#L709>) is used to take a reference to a file (i.e. bump a reference count), based on its file descriptor, and to return the `struct file` pointer for it; references are dropped with [`fput()`](<https://elixir.bootlin.com/linux/latest/source/fs/file_table.c#L329>). Some recent profiling at Watson's employer, Facebook, found the function to be taking a sizable portion of the CPU time for some applications, so he wanted to talk about some of the things he has tried to make that situation better. 

Watson found that `fget()` and `fput()` were taking up to nearly 3% of the CPU for some services that Facebook runs. The worst was 2.9% for an internal service, but the [Memcached](<http://memcached.org/>) distributed key-value store was at 2% and [mcrouter](<https://github.com/facebook/mcrouter>), which is used to scale Memcached, was at 1.7%. Various other services ranged from 0.3% to 0.9%. 

His first thought was that taking up so much CPU simply to manage the reference count is excessive. But he noticed that the services that seemed most affected were networking services, so he guessed that something in the network stack might be causing this. He focused on Memcached and found that 72.5% of the CPU used by `fget()` and `fput()` was coming from the [`recvfrom()`](<http://man7.org/linux/man-pages/man2/recv.2.html>) system call. The other two system calls that used significant amounts of CPU time were [`epoll_pwait()`](<http://man7.org/linux/man-pages/man2/epoll_wait.2.html>) at 11.5% and [`sendmsg()`](<http://man7.org/linux/man-pages/man2/send.2.html>) at 11%. 

[ ![\[Dave Watson\]](https://static.lwn.net/images/2019/lsf-watson-sm.jpg) ](<https://lwn.net/Articles/787510/>)

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

He noted that Memcached is a call-response service; it receives a request for a key's value and sends it back. So `sendmsg()` is being called as often as `recvfrom()` but is contributing much less to the problem. His suspicion is that the receive path is taking a bunch of cache misses, but that the send comes relatively soon after it, so the cache will have fewer misses. 

He then annotated `fget()` and found that cache misses on the file-descriptor table and on dereferencing the `struct file` pointer were taking up much of the CPU time, as does the atomic increment for the reference count. So he tried two different ways to reduce that overhead. 

The first was to delay the reclamation of the `struct file` pointer by not incrementing the reference count in `fget()` (and not decrementing it in `fput()`) for some system calls that use a single file descriptor (e.g. `recvfrom()`, `sendmsg()`, various `epoll*()` calls, etc.). The calls are not likely to block, but if they do, the behavior reverts to the existing path. He worked up a proof of concept for this idea, but the results were underwhelming, so he does not recommend going down that path. 

His second attempt tried to get at the cache misses that were causing much of the CPU use by creating a cache of `struct file` pointers on a per-task basis. Multiple file pointers in the file-descriptor table are sharing the same cache line, when any of those get changed, even by an unrelated thread, there is cache-line bouncing. For Facebook, the file descriptors stay with the same thread once the "accepter" thread hands them off to a processing thread, so the cache eliminates the processor cache misses and the performance loss that went along with them. It is not a fancy cache as, once again, he just wanted to see if it helped. There is a complication, however, as it is not clear how to flush the cache, he said; if you want `close()` to work, though, you need to flush the cache. He ended up adding a field in `struct file` that pointed at the cache entry so that `close()` could do the right thing. 

Overall, his proof-of-concept seems to work well. Most of the overhead from cache misses for the file-descriptor table are gone; that gives a roughly 2x performance increase. He has not thought about handling the cache misses for the `struct file` pointer dereferencing, but that could be next. 

Jan Kara wondered if the file-descriptor table bouncing could be handled by allocating file descriptors in a way that causes separate threads to use different parts of the table. Some applications may depend on sequential file-descriptor allocation, however, which may not mesh well. It would potentially stop the cache-line bouncing of the table, though. Matthew Wilcox suggested that the scheme could be prototyped using [`dup2()`](<http://man7.org/linux/man-pages/man2/dup2.2.html>). 

Wilcox also suggested that moving to a process-based, rather than thread-based, model for these services would be another way to avoid some of the problems that Facebook is experiencing. Watson said that would essentially be impossible. The idea of a per-thread file-descriptor range is worth experimenting with, however.
