---
title: RCU and open file accounting
url: https://lwn.net/Articles/174641/
date: "March 7, 2006"
category: "Read-copy-update"
author: 
---

David Miller has been making great progress in his port of the Linux kernel to Sun's new "Niagara" (SPARC) CPU architecture. He has [run into one little problem](<https://lwn.net/Articles/174642/>), however: 

I just wanted to report that I am hitting the "VFS: file-max limit xxx reached" problem quite easily on my 32-cpu Niagara machine with 16GB of ram with current 2.6.x GIT. It seems far too easy to get a box into this state due to SLAB fragmentation and RCU. And once you get a machine into this state it is totally unusable. 

Our test case is usually a "make -j8192" kernel build along with a parallel bootstrap of gcc. That puts about 256 processes on each cpu's runqueue, I doubt ksoftirqd can run much at all. 

The file limit problem was last discussed here [in October](<http://lwn.net/Articles/156144/>), when it delayed the release of the 2.6.14 kernel. A fix merged at that time made the problem harder to trigger, but, as David's experience shows, the problem has not been solved altogether. One might argue that a relatively small number of users run the sort of workload that David is playing with. But the point remains: with current kernels, including the upcoming 2.6.16 release, it is possible for a suitably-written program to run the open file count to its maximum, thus denying any sort of service to other users. This seems like a problem which one might want to fix. 

One piece of the puzzle here is the way that the open file count is managed. Currently, that count is decremented in the slab destructor set up for `file` structures. This method works, but it can cause the decrement to be delayed by an arbitrary amount of time, with the result that the open file count overstates the number of files which are actually held open by processes in the system. Moving that operation out of the slab destructor can help to keep the count more in sync with reality. 

The core of the problem, however is the use of the read-copy-update (RCU) mechanism for management of `file` structures. When a file is closed, the task of freeing the structure is queued in RCU. Using RCU lets the kernel ensure that the structure is not freed while references to it remain, but without the sort of locking overhead that comes with other techniques. As a result, performance is measurably improved on SMP systems. 

When there is a lot of opening and closing of files going on (such as, say, when a wild-eyed developer starts an 8192-process kernel build), the length of the RCU callback queue can get quite long. By the time that the RCU code decides that the system has quiesced and it is safe to invoke the RCU callbacks, the queue might have thousands of entries. Working through the entire callback queue led to latency problems elsewhere in the system, so 2.6.14 included a patch which put an upper limit on the number of callbacks which would be processed in any single iteration. 

The limit helped with the latency problem. But, if the generation of RCU callbacks continues at a high rate, the length of the callback queue can only grow. Every entry in the queue represents memory which could be returned to the system, but which has not yet been made available. So, as the queue grows, memory gets fragmented and the system heads towards the dreaded out-of-memory state. 

An attempt at a solution can be found in [this patch by Dipankar Sarma](<http://www.hill9.org/linux/kernel/patches/2.6.16-rc3/rcu-batch-tuning.patch>), which has been sitting in the -mm tree for a while. Dipankar's patch puts a configurable upper limit on the number of RCU callbacks which will be processed in any single batch; that allows system administrators to tune the batch size to their particular needs. On a server which is dealing with large number of file requests, and on which latency is not a crucial issue, the batch size can be set to a large number. 

The patch also adds a high-water limit. If the length of the RCU callback queue ever exceeds that limit, the RCU code will (1) set the batch limit to infinity (or the integer representation thereof) and (2) send out an inter-processor interrupt forcing every CPU on the system to schedule. The combination of these actions will cause the system to work through the entire RCU queue at the soonest possible time. Once the queue length goes below a low-water limit, the old batch limit will be restored. 

It is, in other words, a somewhat unsubtle approach; the system is given a kick in the rear and told to go clean up its mess. But, it seems, that is exactly what the system needs at such a time. The cleanup task can only be deferred for so long; the work eventually needs to be done regardless. David has reported that the patches fix the problem on his Niagara system, and suggests that they should be merged into 2.6.16. It is a fairly significant patch to merge at this late point in the cycle, but there seems to be a reasonably high level of confidence in its stability. So, chances are that it will be included as a preferable alternative to shipping 2.6.16 with a known problem.
