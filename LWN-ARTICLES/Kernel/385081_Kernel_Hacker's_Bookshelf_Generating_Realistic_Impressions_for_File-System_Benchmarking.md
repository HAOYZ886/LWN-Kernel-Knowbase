---
title: "Kernel Hacker's Bookshelf: Generating Realistic Impressions for File-System Benchmarking"
url: https://lwn.net/Articles/385081/
date: "April 28, 2010"
category: "Filesystems-Benchmarking; Kernel Hackers Bookshelf"
author: "April 28, 2010 This article was contributed by Valerie Aurora"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

April 28, 2010

This article was contributed by Valerie Aurora

""File systems benchmarking is in a state of disarray."" This stark and undisputed summary comes from the introduction to "[Generating Realistic Impressions for File-System Benchmarking [PDF]](<http://www.cs.wisc.edu/adsl/Publications/impressions-fast09.pdf>)" by [Nitin Agrawal](<http://www.nec-labs.com/~nitin/>), Andrea Arpaci-Dusseau, and Remzi Arpaci-Dusseau. This paper describes Impressions, a tool for generating realistic, reproducible file system images which can serve as the base of new file system benchmarks. 

First, a little history. We, the file systems research and development community, unanimously agree that most of our current widely used file system benchmarks are deeply flawed. The Andrew benchmark, originally created around 1988, is not solely a file system benchmark and is so small that it often fits entirely in-cache on modern computers. Postmark (c. 1997) creates and deletes small files in a flat directory structure without any `fsync()` calls; often the files are deleted so quickly that they never get written to disk. The company that created Postmark, Netapp, [stopped hosting the Postmark code and tech report](<http://communities.netapp.com/thread/1521>) on their web site, forcing developers to pass around bootleg Postmark versions in a bizarre instance of benchmark samizdat. [fs_mark](<http://sourceforge.net/projects/fsmark/>) (c. 2003) measures synchronous write workloads and is a useful microbenchmark, but is in no way a general purpose file system benchmarking tool. [bonnie](<http://www.textuality.com/bonnie/>) (c. 1990) and [bonnie++](<http://www.coker.com.au/bonnie++/>) (c. 1999) tend to benchmark the disk more than the file system. In general, run any file system benchmark and you'll find a file system developer who will tell you why it is all wrong. 

Why has no new general purpose file system benchmark gained widespread use and acceptance since Postmark? A new benchmark is a dangerous creature to unleash on the world: if it becomes popular enough, years of research and development can go into making systems perform better on what could, in the end, be a misleading or useless workload. "No benchmark is better than a bad benchmark," is how the thinking goes, at least in the academic file systems development community. I've seen several new benchmarks quashed over the years for minor imperfections or lack of features. 

However, creating excellent new file systems benchmarks is difficult without intermediate work to build on, flawed though it may be. It's like demanding that architects go straight from grass huts to skyscrapers without building stone buildings in between because stone buildings would be an earthquake hazard. As a result, the file systems benchmarking community continues to live in grass huts. 

#### Impressions: Building better file system images

One thing the file systems community can agree on: We need better file system images to run our benchmarks on - a solid foundation for any putative skyscrapers of the future. The most accurate and reproducible method of creating file system images is to make a copy of a representative real-world file system at the block level and write it back to the device afresh before each run of the benchmark. Obviously, this approach is prohibitively costly in time, storage, and bandwidth. Creating a tarball of the contents of the file system, and extracting it in a freshly created file system is nearly as expensive and also loses the information about the layout of the file system, an important factor in file system performance. Creating all the files at once and in directory order is a best case for the file system block allocation code and won't reflect the real-world performance of the file system when files are created and deleted over time. In all cases, it is impractical for other developers to reproduce the results using the same file system images - no one wants to download (and especially not host) several hundred gigabyte file system images. 

This is where Impressions comes in. Impressions is a relatively small, simple, open-source tool (about 3500 lines of C++) that generates a file system image satisfying multiple sophisticated statistical parameters. For example, Impressions chooses file sizes using combinations of statistical functions with multiple user-configurable parameters. Impressions is deterministic: given the same set of starting parameters and random seeds, it will generate the same file system image (at the logical level - the on-disk layout may not be the same). 

#### Impressions: The details

The directory structure of the file system needs to have realistic depth and distribution of both files and directories. Impressions begins with a directory creation phase that creates a target number of directories. The directories are distributed according to a function of the current number of subdirectories of a particular directory, based on a 2007 study of real-world file system directory structure. A caveat here is that creating the directories separately from the files will not properly exercise some important parts of file system allocation strategy. However, in many cases most of the directory structure is static, and most the changes occur as creation and deletion of files within directories, so creation of directories first reflects an important real-world use case. 

The distribution of file sizes can't be accurately modeled with any straight-forward probability distribution function due to a second "bump" in the file size distribution, which in modern file systems begins around 512MB. This heavy tail of file size distribution is usually due to video files and disk images, and can't be ignored if you care about the performance of video playback. Impressions combines two probability distribution functions, a log-normal and a Pareto, with five user-configurable parameters to produce a realistic file size distribution. 

Files are assigned a depth in the directory tree according to "an inverse polynomial of degree 2." Whatever that is (the code is available for the curious), Figure 2(f) in the paper shows that the resulting distribution of files by namespace depth is almost indistinguishable from that in a real-world file system. Impressions also supports user-configurable "Special" directories with an exceptionally large number of files in them, like `/lib`. 

The authors of Impressions clearly understood the importance of realistic file data; the example use case in the paper is performance comparison of two desktop search applications, which depend heavily on the actual content of files. Filling all files with zeroes, or randomly generated bytes, or repeats of the same pieces of text would make Impressions useless for any benchmark that depends on file data, such as those testing file system level deduplication or compression. Impressions supports two modes of text file content generation, including a word popularity model suitable for evaluation of file search applications. It also creates files with proper headers for sound files, various image and video formats, HTML, and PDF. 

Generation of file names is rudimentary but includes advanced support for realistic file name extensions, like `.mp3`. The file name itself is just a number incremented by one each time a file is created, but the extension is selected from a list of popular extensions according to percentiles observed in earlier file system studies. Popular extensions only account for about half of file names; the rest of the extensions are randomly generated. 

One case in which file names generated this way won't be useful is in evaluating a directory entry lookup algorithm. Sequential search of a directory for a particular directory entry isn't very efficient. Instead, most modern file systems have some way to quickly map a file name to its location in a directory, usually based on a hash of the characters of the file name. This mapping function may be more or less efficient on Impression's sequential numerical file names compared to real-world names. File name length also influences performance, since it changes the number of directory entries that fit in a block. Overall, file name generation in Impressions is good enough, but there are opportunities for improvement. 

One of the most important features of Impressions is its support for deliberate fragmentation of the file system. Impressions creates fragmentation by writing part of a file, creating a new file, writing another chunk of the file, and then deleting the new file. This cycle is repeated until the requested degree of fragmentation is achieved. Note that file systems with good per-file preallocation may never fragment in this scheme unless the disk space is nearly full or no contiguous free stretches of disk space are left. In this case, fragmenting a file system to the requested degree may take a while. More efficient methods of fragmenting a file system might be necessary in the future. Impressions could also use FIBMAP/FIEMAP to query the layout of file systems in a portable manner; currently calculation of the "fragmentation score" is only supported on ext2/3. 

An interesting feature described in the paper but not available in the version 1 release of Impressions is support to run a specified number of rounds of the fragmentation code - sort of a fragmentation workload. This will show the difference in disk allocation strategies between file systems. For example, if one file system manages allocation well enough that it normally never exceeds 30% discontiguous blocks, and the other normally always exceeds 60% discontiguous blocks, it doesn't always make sense to compare their performance when they are both at 50% discontiguous blocks. Instead, running a set fragmentation workload would result in different "natural" fragmentation levels in both file systems, providing a more realistic baseline for performance comparison. 

#### Impressions development

Impressions is open sourced under the GPLv3 and downloadable [here.](<http://www.cs.wisc.edu/adsl/Software/Impressions/>) The original author, Nitin Agrawal, has graduated (now at NEC Labs) and does not currently have plans for developing Impressions further. This is a rare golden opportunity for a new maintainer to work on an influential, high-profile project. The code is, in my opinion, easy to understand and clearly written (although I've spent the last year working on e2fsprogs and the VFS, so take that with a grain of salt). Some open areas for contribution include: 

  * Measure actual fragmentation using FIBMAP/FIEMAP 
  * Smarter filename generation 
  * Addition of hard links and symbolic links 
  * Performance improvement 
  * Scaling to larger file systems (> 100GB) 
  * Packaging for distributions 
  * More robust error checking and handling 
Another possibility for future development is Lars Wirzenius's [`genbackupdata`](<http://braawi.org/genbackupdata/
>) tool, written in Python. The goal of this tool is to generate a representative file system image for testing a backup tool. It already has some of the features of Impressions and others appear to be easy to add. Python may be a more suitable language for long-term development and maintenance of a file system creation tool. 

#### Conclusion

Impressions is an enormous step forward in the file system benchmarking world. With a little polishing and a dedicated maintainer, it could become the de facto standard for creating file systems for benchmarking. Impressions can report the full set of parameters and random seeds it uses, which can then be used for another Impressions run to recreate the exact same logical file system (actual layout will vary some). Impressions can be used today by file system and application developers to create realistic, reproducible file system images for testing and performance evaluation.
