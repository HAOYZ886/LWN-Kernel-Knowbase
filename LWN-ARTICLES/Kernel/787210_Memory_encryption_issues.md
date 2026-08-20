---
title: Memory encryption issues
url: https://lwn.net/Articles/787210/
date: "May 1, 2019"
category: "Memory management-Memory encryption"
author: "By Jonathan Corbet May 1, 2019 LSFMM"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 1, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

"People think that memory encryption sounds really cool; it will make my system more secure so I want it". At least, that is how Dave Hansen characterized the situation at the beginning of a session on the topic during the memory-management track at the 2019 Linux Storage, Filesystem, and Memory-Management Summit. This session, also led by Kirill Shutemov, covered a number of aspects of the memory-encryption problem on Intel processors and beyond. One clear outcome of the discussion was also raised by Hansen at the beginning: users of memory encryption need to think hard about where that extra security is actually coming from. 

Memory encryption takes a lot of forms, Hansen said; he was prepared to talk mostly about Intel's offerings, but he didn't want the discussion to be limited to that. The feature is driven by a desire for the protection of data while it is at rest. Data stored in ordinary RAM is normally thought of as disappearing when the power goes away, but even that data can be [![\[Dave Hansen\]](https://static.lwn.net/images/conf/2019/lsfmm/DaveHansen-sm.jpg)](<https://lwn.net/Articles/787212/>) recovered with a sufficiently sophisticated offline attack. Persistent memory, of course, makes such attacks easier. Those devices can have hardware locks, but they apply to the whole device, while users would like to have more fine-grained protection. 

In particular, there is a desire to create separate encryption domains within a system — one key per container, essentially. Users feel that it makes them secure, he said. This type of encryption might be able to protect users from a compromised operating system, but that is not something we can do now. 

On the Intel side, effort is being put into the ["multi-key total-memory encryption" feature](<https://lwn.net/Articles/776688/>) (MKTME). The older TME functionality only supports a single key for the entire system; it provides protection for data at rest, but does not allow for any sort of separation of domains within the system. MKTME changes that by allowing different processes to run with different keys. There are patches implementing MKTME support (for anonymous memory only, initially) out for consideration. 

Rik van Riel asked at this point whether the encryption is managed within the CPU; in particular, is data stored in the CPU caches encrypted? The answer was "no"; encryption is implemented in the memory controller. Thus, Van Riel continued, it provides no protection against attacks exploiting vulnerabilities like [L1TF](<https://lwn.net/Articles/762570/>). Hansen agreed, noting that technologies like memory encryption should always be evaluated against prior vulnerabilities to see what they would have protected against. Memory encryption does not help against speculative-execution attacks. 

Hansen raised another interesting problem that doesn't seem to have an easy solution. There are a number of attacks on encryption keys that are helped by having large amounts of encrypted data available. When a mechanism like MKTME is in use, the memory controller essentially becomes a high-bandwidth encryption engine; that could facilitate such attacks. Such things need to be taken into account when considering memory encryption to be sure that it provides a real security benefit. 

Somebody asked whether memory encryption works with DMA I/O; the answer is that yes, it can be done. But supporting DMA requires programming the encryption key into the IOMMU. The AMD memory-encryption implementation does bounce-buffering instead for now. 

Future work, Hansen said, includes implementing support for the protection of file-backed memory and persistent memory. It should eventually be possible to set keys at a device level, or on individual files (or directory trees, more likely). This functionality may be built on the fs-crypt feature supported by some filesystems now. 

Van Riel asked how many keys can be supported by current systems; the answer was that each memory controller can handle up to 64 different keys. In the current patches, all controllers must be configured with the same keys. There is talk of breaking that link, so that different controllers could have different keys, thus increasing the total number of key slots available but, Hansen said, "that sounds like a nightmare" to implement. Memory can normally be allocated arbitrarily across controllers; if only some controllers could handle a given process's encryption key, though, the situation would be complicated immensely. It might make sense when data is tied to a specific controller — data on a specific persistent-memory device, for example. 

Shutemov talked briefly about the additional challenges posed by CPU hotplugging. A new CPU likely brings with it a new memory controller, which must then be programmed with the same keys used by the existing controllers. To be able to do that, though, the kernel must keep a copy of [![\[Kirill Shutemov\]](https://static.lwn.net/images/conf/2019/lsfmm/KirillShutemov-sm.jpg)](<https://lwn.net/Articles/787213/>) the keys in its own memory, where an attacker may try to steal it. It is not possible (or not intended to be possible, anyway) to extract the keys from the memory controllers themselves, so if the kernel can delete its copy of the keys after setting them, key-stealing attacks should be nearly impossible. Storing the key in the kernel, thus, can only reduce the security of the system as a whole. 

As a result, there is still discussion over whether the MKTME patches should allow users to set their own keys for anonymous memory at all. There does not appear to be any security benefit from doing so; indeed, the opposite seems to be true. Users would be more secure without that feature. One potential benefit to user-supplied keys for anonymous memory, though, could be to provide a sort of secure-erase feature. When a given user's processes shut down, the associated encryption key can be overwritten in the memory controllers and deleted from the kernel, after which that user's data will be inaccessible. 

Matthew Wilcox noted that encryption doesn't come for free; he wondered about what the power cost of memory encryption might be. Nobody had a good answer to that question. Hansen did note that there is a "single-digit" percentage increase in memory latency when encryption is used; total memory bandwidth is unaffected, but latency does increase. Another cost comes when keys are updated; that requires flushing all of the CPU caches, which is expensive. Setting keys is a privileged operation in the current patches; key slots are a limited resource, so ordinary users should not be allowed to consume them at will. 

As the session ran out of time, it began to wander a bit. A final question had to do with where the key used for TME (which must already be present at boot) comes from — how do users know that it is secure? There was not a clear answer to that question either, but Hansen did note that the key originates in the CPU. If the CPU itself cannot be trusted, then the question of how the encryption key is generated does not matter much.
