---
title: Supporting NFS v4.2 WRITE_SAME
url: https://lwn.net/Articles/1025257/
date: "June 16, 2025"
category: "Filesystems-NFS"
author: "By Jake Edge June 16, 2025 LSFMM+BPF"
---

By **Jake Edge**  
June 16, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

At the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF), Anna Schumaker led a discussion about implementing the NFS v4.2 [`WRITE_SAME` command](<https://datatracker.ietf.org/doc/html/rfc7862#section-15.12>) in both the NFS client and server. `WRITE_SAME` is meant to write large amounts of identical data (e.g. zeroes) to the server without actually needing to transfer all of it over the wire. In her [topic proposal](<https://lwn.net/ml/all/f9ade3f0-6bfc-45da-a796-c22ceaeb4722%40oracle.com/>), Schumaker wondered whether other filesystems needed the functionality, so that it should be implemented at the virtual filesystem (VFS) layer, or whether it should simply be handled as an NFS-specific [`ioctl()`](<https://www.man7.org/linux/man-pages/man2/ioctl.2.html>). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The NFS `WRITE_SAME` operation was partly inspired by the [SCSI `WRITE SAME` command](<https://linux.die.net/man/8/sg_write_same>), she began; it is ""intended for databases to be able to initialize a bulk of records all at once"". It offloads much of the work to the server side. So far, Schumaker has been implementing `WRITE_SAME` with an `ioctl()` using a structure that looks similar to the [application data block structure](<https://www.rfc-editor.org/rfc/rfc7862#section-8.1.1>) defined in the [NFS v4.2 RFC](<https://www.rfc-editor.org/rfc/rfc7862>) for use by `WRITE_SAME`. 

[ ![\[Anna Schumaker\]](https://static.lwn.net/images/2025/lsfmb-schumaker-sm.png) ](<https://lwn.net/Articles/1025410/>)

On the server side, it would make sense to have a function that gets called to process the `WRITE_SAME` command, but it would be nice if that same function was available to clients; they could use it as a fallback when the server does not implement `WRITE_SAME`. Other filesystems could potentially also use the functionality, either with the SCSI `WRITE SAME` or for some other filesystem-specific use case. 

The application data block allows for `WRITE_SAME` commands that write various patterns to the storage, but Christoph Hellwig suggested that all of that complexity should be avoided. He was responsible for writing the `WRITE_SAME` definition for NFS and for killing off the Linux block-layer support for the SCSI `WRITE SAME` patterns; ""don't do it"", he said with laugh. `WRITE_SAME` for zeroing is ""perfectly fine"", SCSI supports that, but ""exposing all the detailed, crazy patterns"" is ""not sane"". Getting the semantics right for all of the different cases is extremely difficult. Schumaker said that sounded reasonable. 

There is already an API available for clients to use, Amir Goldstein said: [`fallocate()`](<https://www.man7.org/linux/man-pages/man2/fallocate.2.html>) with the `FALLOC_FL_ZERO_RANGE` flag. Schumaker said that NFS did not have support for that flag, but Goldstein said that support could be added as the way to provide access to `WRITE_SAME`. Hellwig said that there was a [patch set](<https://lwn.net/ml/all/20250318073545.3518707-1-yi.zhang@huaweicloud.com/>) that he had not yet looked at closely to add an `FALLOC_FL_WRITE_ZEROES` flag that would force the zeroes to be written; it might be a better API for `WRITE_SAME`. That [series is now on v5](<https://lwn.net/ml/all/20250604020850.1304633-1-yi.zhang@huaweicloud.com/>) and seems to be progressing toward inclusion. 

Matthew Wilcox wondered whether only being able to write zeroes would make the `WRITE_SAME` feature less than entirely useful; he remembered a ""a certain amount of pushback because databases need a specific pattern"". There was a fair amount of joking about which of the two Oracle databases (the other being MySQL) he meant; Wilcox works for Oracle, as does Schumaker, who seemed to indicate that she had talked to the MySQL group. In the end, someone seemed to sum up that only supporting zeroing is reasonable: ""zeroes are good"". 

Chuck Lever, who also works for Oracle, said that he had spoken to the Oracle database group. That database does not use the Linux NFS client, so the group did not care about support for `WRITE_SAME` in the client. The group's concern was mostly about support for `WRITE_SAME` in proprietary NFS servers, he said. Wilcox asked: ""and Linux NFS servers?"" Lever said that Oracle databases do not deploy on systems that use those.
