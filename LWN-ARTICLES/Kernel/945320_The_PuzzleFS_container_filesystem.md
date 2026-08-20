---
title: The PuzzleFS container filesystem
url: https://lwn.net/Articles/945320/
date: "September 25, 2023"
category: Filesystems
author: "By Jonathan Corbet September 25, 2023 Kangrejos"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 25, 2023

* * *

[Kangrejos](<https://lwn.net/Archives/ConferenceByYear/#2023-Kangrejos>)

The last year or so has seen the posting of a few new filesystem types that are aimed at supporting container workloads. PuzzleFS, presented at the 2023 [Kangrejos](<https://kangrejos.com/>) gathering by Ariel Miculas, is another contender in this area, but it has some features of its own, including a novel compression mechanism and an implementation written in Rust. 

PuzzleFS, Miculas began, is an immutable (and thus read-only) filesystem that shares design goals with the [Open Container Initiative (OCI) v2 image specification](<https://github.com/opencontainers/image-spec>). It uses content-defined chunking (discussed further shortly) and a content-addressed data store, with file data and metadata stored separately from each other. The project was started by Tycho Andersen in 2021 as an attempt to create a successor to [atomfs](<https://github.com/anuvu/atomfs>). 

The first version of the OCI image specification, he said, had a number of problems, many of which are described in [this 2019 blog post](<https://www.cyphar.com/blog/post/20190121-ociv2-images-i-tar>) by Aleksa Sarai. At the base of those problems is the dependence on tar archives to hold the layers in the filesystem. Tar, as it turns out, is not well suited to the container filesystem problem. 

The format, he said, is defined poorly. It has no index; instead, there is just a header leading directly into the content. The compression mechanism used means that the filesystem image is not seekable; as a result, the entire filesystem must be decompressed even to extract one little file. There is no deduplication; even a small change means re-downloading the entire thing, though layers can be used to work around that problem to an extent. It is machine-dependent, in that directory entries can be shown in a different order on different systems. The lack of a canonical representation has led to a lot of extensions, many of which are solving the same problem. 

PuzzleFS is intended to solve these problems. A filesystem image itself consists of a set of files placed on an underlying filesystem. As with the OCI image format, there is a top-level `index.json` file that contains a set of tags, each of which represents a versioned filesystem and points to a manifest file. The manifest file, in turn, points to an image configuration and the data stored in the actual image layers. Everything else is stored as a set of blobs in the `blobs/sha256` directory. 

Most data in the filesystem is broken into variable-size chunks, then stored as blobs using the SHA256 hash of the content as the file name. The chunking itself is done with the [FastCDC algorithm](<https://www.usenix.org/conference/atc16/technical-sessions/presentation/xia>), which finds "cut points" where a data stream can be split into blobs of varying sizes. Any given stream (the contents of a file, for example) might be split into five or 50 chunks, depending on how those cut points are determined; each chunk then lands as a separate blob under `blobs/sha256`, and its hash is added to the manifest. 

The cut-point algorithm itself uses a sliding-window technique. The data is stepped through, byte by byte, and the hash of the last 48 bytes (for example) is calculated. If the N least-significant bytes of that hash are zero, then a cut point has been found; the data to that point is separated into a separate blob and the process starts anew. 

This algorithm has some interesting characteristics, perhaps most notably its ability to perform deduplication and compression. Since each chunk is stored using its hash as the file name, chunks that are common to multiple files will automatically be shared. In traditional schemes, an update to a file will cause the entire new file to be stored; this is especially true if any bytes are inserted or deleted. Inserting a single byte into a traditionally compressed file will make the entire file after the insertion look different. With content-defined chunking, only the chunk containing the change will differ, while the rest of the file will contain the same chunks, perhaps at a different offset. 

The results can be seen in an experiment that Miculas carried out. He downloaded ten different versions of Ubuntu 22.04 from the Docker Hub; they required 766MB to store in that form. Putting them into the OCI image format with compression reduced that size to 282MB. Placing them all into a PuzzleFS instance, instead, reduced the size to 130MB — without using compression. Adding compression cut the whole thing down to 53MB, a savings of 93% from the original size. 

A goal of PuzzleFS was to always provide a canonical representation of the filesystem. Thus, for example, the traversal order of the source from which it is generated is defined, with both directories and extended attributes being sorted lexicographically. Another goal was direct mounting support. With tar-based formats, the files must first be extracted to an on-disk representation, creating a window where things could be changed before mounting the image. Thus, there is no guarantee that the files seen by the kernel are the same as those that were in the tar archive. PuzzleFS doesn't have that extraction step, so that problem does not exist. 

Data integrity is an important objective in general. It is not possible to use [dm-verity](<https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/verity.html>) in this case to protect the whole volume; while the filesystem is immutable, the underlying data store is not, since adding a new version or layer requires the ability to add new data. So, instead, [fs-verity](<https://www.kernel.org/doc/html/latest/filesystems/fsverity.html>) is used to verify the integrity of the individual files in the data store. When mounting a specific image, the hash of the manifest of interest is given to the `mount` for verification. 

An important objective behind this project was the avoidance of memory-safety bugs. For that reason, the filesystem implementation has been written in Rust. That choice has, he said, removed a lot of pain from the development process. 

There is a working FUSE implementation, and a [kernel implementation](<https://lwn.net/ml/linux-kernel/20230726164535.230515-1-amiculas@cisco.com/>) in a proof-of-concept state. The kernel side depends on a set of filesystem-interface abstractions that are being developed separately, and which should be headed toward the mainline at some point. Some other work is needed to get other dependencies, including the [Cap'n Proto library](<https://capnproto.org/>) used for metadata storage, into proper shape for the kernel. The work is progressing, though; interested folks can find the current code in [this repository](<https://github.com/project-machine/puzzlefs>). 

One topic Miculas did not address is the resemblance between PuzzleFS and [composefs](<https://lwn.net/Articles/917097/>), which shares some similar goals. Composefs has [run into difficulties](<https://lwn.net/Articles/933616/>) getting into the mainline kernel — though other changes intended to support those goals are going in. PuzzleFS has some features that composefs lacks; whether those will be enough to make its upstream path easier is unclear. 

See [the slides from this talk](<https://kangrejos.com/2023/PuzzleFS.pdf>) for more information and details of the on-disk format. 

[Thanks to the Linux Foundation for supporting our travel to this event.]
