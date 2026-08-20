---
title: "Popcorn Linux pops up on linux-kernel"
url: https://lwn.net/Articles/819237/
date: "May 5, 2020"
category: Academic systems; Clusters; Popcorn Linux
author: "By Jonathan Corbet May 5, 2020"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 5, 2020

The end of April saw the posting of [a complex patch set](<https://lwn.net/ml/linux-kernel/cover.1588127445.git.javier.malave@narfindustries.com/>) called "Popcorn Linux distributed thread execution". It is the first appearance on the kernel mailing lists of an academic project (naturally called [Popcorn Linux](<http://popcornlinux.org/>)) that has been underway since 2013 or so. This project has, among other goals, the objective of turning a tightly networked set of computers into something that looks like a single system — a sort of NUMA machine with even larger than usual inter-node costs. The posted code, which is a portion of the larger project, is focused on process migration and memory sharing across machines. It is an interesting proof of concept, but one should not expect to see it merged in anything close to its current form. 

Each node in a Popcorn system is a separate Linux host sitting on the network. Popcorn itself is started by loading a kernel module that is charged with connecting the larger system together. The module reads a list of IP addresses (IPv4 only) directly from a file (`/etc/popcorn/nodes` by default). Each machine will make a TCP connection to every node listed ahead of itself in this file, then wait for an incoming connection from every node listed afterward. Thereafter, each node is known by an integer ID which is simply its position in the `nodes` file. 

There is a hard-coded maximum of 62 nodes. No sort of authentication is done for incoming node connections, which might seem like a bit of a security issue; indeed, the patch set warns against running Popcorn on machines connected to the Internet. There does not seem to be any provision for nodes going up or down or being absent entirely. Comments in the patch set say that the TCP-based communication system ""is intended for Popcorn testing and development purposes only"", suggesting that, someday, somebody will get around to implementing something better. 

#### System calls

Unsurprisingly, some new system calls are needed to allow applications to work within the larger Popcorn Linux system. To start with, an application can query the set of available nodes with this new system call: 

```
struct popcorn_node_info {
    	unsigned int status;
    	int arch;
    	int distance;
        };
    
        int popcorn_get_node_info(int *my_nid, struct popcorn_node_info *nodes, int len);
```

On return, `my_nid` will contain the ID of the node the caller is running on, and `nodes`, an array of `len` `popcorn_node_info` structs indexed by node ID, will be filled in with the details of the first `len` nodes (up to the number that actually exist, of course). The `status` field in each entry will be zero if the corresponding node is offline, one otherwise. The `arch` field will describe the node's architecture (`POPCORN_ARCH_X86` in the current patch set, since x86 is the only supported architecture), and `distance` is always zero. 

The system call to move the current thread to another node is: 

```
int popcorn_migrate(int node_id, void *uregs);
```

where `node_id` identifies the node to which the thread should be moved, and `uregs` is, for some reason, the contents of the processor registers to be restored when the thread resumes on the new node. The passing of the processor registers separately might be an artifact of a Popcorn feature that is not part of this patch set: the ability to move threads to remote nodes with a different processor architecture. In the posted patch set, threads can only move themselves; the underlying code is written to allow other processes to force a thread to move, though. 

While `popcorn_migrate()` looks like a general facility to move threads around, in practice it seems to be a bit more limited than that. A moved ("remote") thread retains a connection to its "origin" node; indeed, the original thread is still present on the origin node, it is just prevented from executing while the remote thread is running. A remote thread can only be moved back to the origin, so migrating a thread between two remote nodes would be a two-step operation, first moving it back to the origin then to out the new node. 

The current execution status of a thread can be had with the last of the new system calls in this patch set: 

```
struct popcorn_thread_status {
    	int current_nid;
    	int peer_nid;
    	pid_t peer_pid;
        };
    
        int popcorn_get_thread_status(struct popcorn_thread_status *status);
```

This call will fill in `status` with the current node ID for the calling thread, the other node it is connected to, and its process ID on that node. If the thread is not currently migrated, both `current_nid` and `peer_nid` will be the ID of the origin node. 

#### Supporting remote threads

Once a thread has been moved to another node, more work must be done to keep things synchronized. For example, if the remote thread exits, the origin thread must be made to exit too. A signal sent to the origin thread must be propagated to the remote version where the work is actually being done. All of this is handled by intercepting various actions and sending messages across the inter-node connections to cause the right things to happen. Some especially complicated code appears to be making futexes work across machines. 

Migrating a thread sets up the basic information it needs to run, but leaves a lot of stuff behind; in particular, almost the entirety of the thread's memory-layout information still lives on the origin node, where it might well be shared with other threads. It is not surprising that memory management is the focus of some of the most complex code in the Popcorn patch set. 

The set of virtual memory areas (VMAs) describing the thread's address-space layout will be shared with any other threads running in the same process — threads that probably have not been migrated to the same target node. So, while the migrated thread needs to mirror that VMA arrangement, it has little ability to change it without coordinating with the origin node. For VMAs, this coordination is handled by actually executing almost all operations at the origin. 

Thus, for example, if the migrated thread calls [`mmap()`](<http://man7.org/linux/man-pages/man2/mmap.2.html>), that call will be intercepted and shipped back to the origin for execution. The origin node will send back a response describing the result of the operation; the migrated thread's memory layout will then be adjusted to match. Other calls, including `[brk()](<http://man7.org/linux/man-pages/man2/brk.2.html>)` and [`madvise()`](<http://man7.org/linux/man-pages/man2/madvise.2.html>) are handled in the same way. 

Pages of actual memory need to be handled a bit differently, though, or performance will suffer horribly. Popcorn implements a protocol to allow the ownership of pages to move between nodes, much like ownership of cache lines can move between processors. Read-only copies of pages can be spread across a set of nodes, but only one node can be modifying a specific page at any given time. Much of the coordination is handled, once again, by the origin node, which handles tasks like sending and receiving copies of pages, invalidating pages on remote nodes, revoking page ownership, and more. 

The patch set also adds a new `madvise()` operation, `MADV_RELEASE`, which explicitly releases a remote node's ownership of a range of pages. 

#### Will it pop soon?

There is a lot more to Popcorn Linux than what has been posted to the list so far. There is a mechanism to run multiple kernels on the same machine, for example, using a modified version of the kexec mechanism. There is a whole [fault-tolerance project](<http://popcornlinux.org/index.php/ft-linux>) underway. There is [a mechanism](<http://popcornlinux.org/index.php/hexo>) to offload low-demand virtual machines to slow (but power-efficient) embedded boards, possibly running a different processor architecture. And more; the web site is well-populated with academic papers describing various parts of the system. 

Popcorn Linux seems like an interesting project, so readers unfamiliar with how kernel development works may be surprised to see that this patch set, posted on April 29 and which has received a fair amount of attention on various Internet sites since, has not seen a single response on the mailing list. The reason for that is relatively straightforward, though: what has been posted is a pile of code, rather than a patch series that is intended for serious review and consideration. Patch 1 introduces the system calls, for example, but the structure definitions they rely on don't show up until Patch 5, and the messaging infrastructure, without which nothing works, shows up last. Your editor can attest that reading a patch series organized in this way is not a simple task; many busy kernel developers are unlikely to try. 

One often hears complaints that the work done in academic settings almost never makes it into Linux; this seems paradoxical, given that the open development process behind Linux should be a natural fit for academic developers. This patch set shows where the roadblocks are, though. It represents many years worth of work, but none of that work was directed toward creating a patch set that is ready to be considered for merging. 

To get this work even considered for upstream, the Popcorn Linux developers will have do a number of things. The patches will have to be reworked into a bisectable series, where each patch stands alone and can be considered on its own merits. The temporary messaging system will have to be replaced with something that is robust, secure, and fast. Performance benchmarks will have to be prepared, including the impact of Popcorn Linux on systems that are not using any of its features. Documentation is distressingly optional in kernel development, but a few pages of introductory material might help developers review the patches. And so on. 

Doing all of that would be a lot of work, even before one gets into the code changes that are likely to become necessary during the review process. This work is expensive and is unlikely to lead to the publication of even a single thesis or academic paper. It is unsurprising that getting code of this complexity upstream tends to look unappealing to academic researchers. So that work is rarely done. 

Sadly, that seems likely to be the fate of Popcorn Linux as well, unless somebody can come up with the funding and the motivation to make it suitable for the decidedly non-academic Linux kernel. Even if it is not merged, though, Popcorn Linux may eventually inspire some energetic developer to adopt some of its best ideas and get them upstream in some form. There is a lot of interesting work to be found in this project; hopefully some of it will eventually graduate from the academic setting and onto our systems.
