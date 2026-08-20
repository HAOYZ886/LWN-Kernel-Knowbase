---
title: The problem of unnecessary readahead
url: https://lwn.net/Articles/1016860/
date: "April 18, 2025"
category: "Memory management-Readahead"
author: "By Jonathan Corbet April 18, 2025 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 18, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The final session in the memory-management track of the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit was a brief, last-minute addition run by Kalesh Singh. The kernel's readahead mechanism is generally good for performance; it ensures that data is present by the time an application gets around to asking for it. Sometimes, though, readahead can go a little too far. 

Singh had a couple of example cases; the first had to do with ELF binaries that include extra padding for compatibility with systems that have a large base-page size. On systems with smaller pages, that padding is just useless data that could happily remain on disk, but the readahead machinery is reading it into the page cache anyway. That creates useless I/O and wastes memory with data that nobody will need. Another example was APK (Android package) files, which contain a combination of compressed and uncompressed data in the same file. Readahead is bringing in the compressed data, even though it will never be read. 

[![\[Kalesh Singh\]](https://static.lwn.net/images/conf/2025/lsfmm/KaleshSingh-sm.png)](<https://lwn.net/Articles/1016861/>) In both cases, Singh said, readahead is bringing in unwanted data, even if it is outside of the part of the file that has been mapped. [`fadvise()`](<https://man7.org/linux/man-pages/man2/posix_fadvise.2.html>) can be used to prevent the readahead, but that affects the entire file, including the parts where readahead is beneficial. Singh has also tried placing [guard pages](<https://lwn.net/Articles/1011366/>) in the memory area where the file is mapped, but they do not stop readahead. David Hildenbrand suggested that having the readahead mechanism actively look for guard regions would be a good idea, but this checking might prove to be expensive. 

A participant asked why the whole file, including the padding, is being mapped; Singh said that the alternative would create a lot more virtual memory areas (VMAs), which would have a performance cost of its own. Hildenbrand suggested that [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) could be used to force the reading of the desired part of the file. The memory of that operation would be lost, though, if the populated pages are reclaimed. This approach could also slow down application load time, he said. 

Singh had one other possibility to discuss: optimizing the handling of sparse files. The holes in files could be marked in the kernels and mapped to the zero page if accessed by the process. There would be some pitfalls; there are complications if the zero-page is modified (causing a copy-on-write operation) while pinned into memory for example. It would also complicate the handling of files that have been mapped shared; perhaps, he said, that case does not need to be supported. 

As the session (and the conference) ended, Hildenbrand suggested that efforts to optimize for unexpected cases would be misplaced. Perhaps, he said, it would be sufficient to just stop readahead whenever a hole is reached in the file. 

Singh has [posted the slides](<https://drive.google.com/file/d/1MOJu5FZurV4XaCLrQhM9S5ubN7H_jEA8/view?usp=drive_link>) from this session.
