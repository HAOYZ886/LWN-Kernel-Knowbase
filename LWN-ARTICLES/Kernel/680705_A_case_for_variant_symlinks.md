---
title: A case for variant symlinks
url: https://lwn.net/Articles/680705/
date: "March 23, 2016"
category: Filesystems
author: "March 23, 2016 This article was contributed by Neil Brown"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

March 23, 2016

This article was contributed by Neil Brown

Variant symlinks are symbolic links that behave differently depending on details of the process that reads or follows the link. They have a history going back at least to the 1980s when various vendors of Unix systems wanted to be compatible with both BSD Unix from UCB (The University of California at Berkeley), and System V Unix from AT&T. Details varied, but the core idea was that some attribute of a process could be used to modify the target of a symlink or to select among multiple options. This would allow, [for example](<https://en.wikipedia.org/wiki/Symbolic_link#Variable_symbolic_links>), some processes to see `/bin` as a symlink to `/.ucbbin`, while others would see `/.attbin`.

While those issues are long behind us, the desire for variant symlinks still pops up from time to time, most recently in a [proposal](<http://thread.gmane.org/CACsf_wyTqDyfwVnJeWC3ExO5Jeh5mjKxQP2HmJF6o9wuSXATaA@mail.gmail.com>) by Cole Minnaar for a "Variant Symlink Filesystem". The proposed filesystem — currently implemented as an [out-of-tree kernel module](<https://github.com/onslauth/varsymfs>) — takes an extremely simple approach to the problem. The filesystem provides a single directory that contains a single symlink called `resolve`. When any process reads or follows this link, the filesystem looks though that process's environment for a particular environment variable, specified when the filesystem is mounted, and reports the value of that variable as the content of the symlink.

To use this you would mount a filesystem at some well known location and create links that pass though that location. For example

```
# mount -t varsymfs -o UNIVERSE none /.universe
        # ln -s /.universe/resolve/bin /bin
```

Then: 

```
$ UNIVERSE=/att ls -lL /bin
```

would show the contents of `/att/bin`, while: 

```
$ UNIVERSE=/ucb ls -lL /bin
```

would show those of `/ucb/bin`.

The responses to this proposal were pretty much as would be expected: Minnaar was told that he should use a [FUSE](<https://lwn.net/Articles/68104/>) filesystem written in user space, or use [mount namespaces](<https://lwn.net/Articles/531114/>) to give different processes a different view of the system. Minnaar made it clear that this wasn't just a new idea with no history, but was something he has been working on for some time. Both those ideas had been tried and found wanting.

The problem with a FUSE-based solution is performance. Though the special filesystem is not used for any filesystem I/O and is only needed to look up a single symbolic link, a performance decrease can still be measured. By its nature, a variant symbolic link cannot be cached in the VFS layer, so every request would need to go to user space and back into the kernel. [Recent work](<https://lwn.net/Articles/650786/>) has made symlink lookup largely lockless because, for some workloads, even requiring spinlocks for following a symlink can be too expensive. This was found to be particularly true when compiling code, since searching for include files generates lots of filename lookups. Minnaar identified compilation as a problematic case for FUSE-based variant symlinks too, and even the cost of that spinlock — sufficient to justify a rewrite of the symlink lookup code — is tiny compared to the cost of scheduling a user-space process to provide an answer.

The situation with mount namespaces brings its own set of problems, though of a very different kind. A large part of the focus on namespaces has been the creation of containers to contain processes — once in a container, the process shouldn't be able to get out. Minnaar is not interested in that side at all. He is interested in convenience rather than containment. 

The example he sketched was to support multiple versions of packages that require the use of fixed paths. Many packages, such as Perl and Emacs, include a version number in the path names used for finding support files, such as `/usr/lib/perl5/site_perl/5.22.1`. This allows multiple versions to be installed side by side. Many other packages are not so enlightened, allowing only one version to be installed at a time. It would be possible to fix such packages to support parallel installations, but it seems it was easier to implement variant symlinks. That way each package can behave as though it owns the standard path names and each user can select their preferred package version by setting up some environment variables. 

When it comes to convenience, filesystem namespaces have two problems, one that was mentioned and one that wasn't — yet. The first problem is that the Unix shell doesn't have a "`chns`" command to change namespaces. While you can certainly use `nsenter`, as David Lang [suggested](<http://mid.gmane.org/alpine.DEB.2.02.1603111440160.1949@nftneq.ynat.uz>), this creates a new shell rather than adjusting the state of the old shell. There is a good reason that `cd` or `chdir` is built into the shell — having it external would be nowhere near as convenient. In the same way, `nsenter` would only be as convenient as `export UNIVERSE=/att` if it was built-in.

The second problem is the inevitable combinatorial explosion that namespaces would cause. If there is only a need to select on one axis, `ucb` or `att`, then namespaces could be made to work. If independently selecting between versions of a dozen packages is needed, then there would be a need for potentially thousands of namespaces, one for each combination. In practice, this explosion may not happen, but the need to construct namespaces on demand might not be the most convenient approach.

While variant symlinks may well be useful, it would help to have a variety of concrete use-cases to examine so that we could see exactly how they would be used and informed implementation choices could be made. It is easy to "bikeshed" some variations, like whether a constant prefix should be provided at mount time so the environment variable values don't need to start with "/". However, such bikeshedding is likely to focus on the inconsequential and miss the essential. What we need, as Al Viro [indicated](<http://article.gmane.org/gmane.linux.kernel/2175270>), is to ask ""the right questions for figuring out what requirements"" there are, so as to determine ""the best way to do it"". Whether anything like that occurs remains to be seen.
