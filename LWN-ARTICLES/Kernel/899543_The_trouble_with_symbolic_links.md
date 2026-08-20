---
title: The trouble with symbolic links
url: https://lwn.net/Articles/899543/
date: "July 7, 2022"
category: Symbolic links
author: "July 7, 2022 This article was contributed by Chris Riddoch"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

July 7, 2022

This article was contributed by Chris Riddoch

At the [2022 sambaXP conference](<https://sambaxp.org/archive>), Jeremy Allison gave a talk titled "The UNIX Filesystem API is profoundly broken: What to do about it?". LWN regulars may recall hints of these talks in a recent [comment thread](<https://lwn.net/Articles/882177/>). He started his talk with the problems that symbolic links ("symlinks") cause for application developers, then discussed how the solutions to the problems posed by symlinks led to substantial increases in the complexity of the APIs involved in working with pathnames.

Allison explained that hard links were the first "interesting addition" to the original Unix filesystem API; unlike symlinks, though, they are not dangerous, and are, in fact, easy to use. A hard link is simply the connection between a directory entry and the inode for the file (or directory) to which that entry refers. Unix systems allow multiple links to any file, but require that the inode and directory entries all reside on the same filesystem. 

By contrast, symlinks contain another path as data, and the kernel transparently operates on the file at that path when system calls like [`open()`](<https://man7.org/linux/man-pages/man2/open.2.html>) or [`chown()`](<https://man7.org/linux/man-pages/man2/lchown.2.html>) are called on the symlink. This seemingly innocuous feature led to the addition of incredible amounts of complexity in the effort to fulfill the needs of programs that need to be aware of whether a pathname contains a symlink or not. Such programs include archival programs like `tar`, file synchronization and transfer programs such as `rsync`, network filesystem servers like Samba, and many more that suffer security problems as a result of not giving sufficient attention to symlinks in pathnames.

The variety of security problems resulting from symlinks can be seen in a [search of CVE entries](<https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=symbolic+link>), which gave Allison 1,361 results when he ran it. These include vulnerabilities that facilitate information disclosure, privilege escalation, and arbitrary file manipulation including deletion, among other attacks. Without discussing any specific CVE in detail, he gave an example of the kind of security problem that can result from symlink-related vulnerabilities.

An application running as root may try to check that `/data/mydir` is a regular directory (not a symlink) before opening the file `/data/mydir/passwd`. In between the time the program does the directory check and the file open, an attacker could replace the `mydir` directory with a symlink to `/etc`, and now the file opened is, unexpectedly, `/etc/passwd`. This is a kind of race condition known as a [time-of-check-to-time-of-use (TOCTOU)](<https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use>) race.

#### Symlinks and complexity

Symlinks were created, Allison theorized, because hard links are restricted to linking within the same filesystem, so only symlinks (which lack that restriction) could be used if an administrator wanted to add new storage media without changing the paths to users' data. He quoted an advertisement for 4.2BSD, which touted, "This feature frees the user of the constraints of the strict hierarchy that a tree structure imposes. This flexibility is essential for good namespace management."

The addition of symlinks led to the `[lstat()](<https://man7.org/linux/man-pages/man2/lstat.2.html>)` system call, which provided the means to identify whether the last component in a pathname is a symlink. This was, unfortunately, insufficient for handling symlinks pointing to directories earlier in the path, he explained. An application could attempt to check each component of the path individually, but not atomically -- another application could make a change to one of the components during this process, leading to security vulnerabilities.

An option to the `open()` system call, `O_NOFOLLOW`, exhibits the same problem as `lstat()`. `O_NOFOLLOW` instructs the system call to fail with `ELOOP` if the last component in the pathname is a symbolic link, but it _only_ checks the last component. The [`realpath()`](<https://man7.org/linux/man-pages/man3/realpath.3.html>) C library function follows symlinks in a path and produces an absolute, canonical pathname that the application can then compare with the original. Allison described this as an appealing but incorrect solution to the problem. Another process could make a change in between the time `realpath()` is called and another function is used to manipulate the file in some fashion. In other words, the same TOCTOU race applies here. 

Allison said that the `openat()` system call was designed as a solution to this problem; it introduces the idea of referring to files with respect to a directory that's indicated by an already-open file descriptor. The only reliable way to identify a file's path is to walk the hierarchy using multiple calls to `openat()`. Everything else would be vulnerable to race conditions.

But Allison also pointed out the flaw in this technique. "You cannot create a new directory with `open()`, you cannot remove a file, unlink a file, or delete a directory with an `open()` call." So, more functions following the pattern of `openat()` had to be created: [`mkdirat()`](<https://man7.org/linux/man-pages/man2/mkdir.2.html>), [`linkat()`](<https://man7.org/linux/man-pages/man2/link.2.html>), [`unlinkat()`](<https://man7.org/linux/man-pages/man2/unlink.2.html>), [`renameat()`](<https://man7.org/linux/man-pages/man2/rename.2.html>), and more. Some are still missing, like `getxattrat()` and `setxattrat()`. Some, like [`fchownat()`](<https://man7.org/linux/man-pages/man2/lchown.2.html>) and [`faccessat()`](<https://man7.org/linux/man-pages/man2/access.2.html>), don't follow the pattern cleanly. 

Allison didn't mince words: "So our original clean and beautiful POSIX filesystem API doesn’t look very clean and beautiful anymore...pathnames as a concept are now utterly broken in POSIX." One could reasonably attribute, in part at least, any perceived bitterness to Allison's struggles with the [long road to a fix for CVE-2021-20316](<https://lwn.net/Articles/884052/>) in Samba.

Because of the talk's focus on the role of symlinks in complicating the Unix pathname API, Allison did not directly raise the point that race conditions involving pathnames can occur even without symlinks. It seems a major source of complexity is the lack of a mechanism for atomically batching together operations that involve walking directories and symlinks to eventually perform some operation on a file.

#### Workarounds

Allison then explained the use of the `O_PATH` flag to `open()`, which will return a file descriptor that is only useful for passing to the `*at()` system calls as the `dirfd` argument. Unfortunately for Samba, file descriptors opened with `O_PATH` cannot be used for reading or writing extended attributes. He found a workaround, demonstrated by a snippet of code that he described as "one of the most beautiful hacks I’ve ever seen, it’s so ugly it makes you want to vomit, but it’s amazing."

```
int fd = openat(dirfd, "file", O_PATH|O_NOFOLLOW);
        sprintf(buf, "/proc/self/fd/%d", fd);
        getxattr(buf, ea_name, value, size);
```

The contents of `/proc/self/fd` are symlinks that represent every file descriptor the process has open. Allison explained the code: "If you print into a buffer '`/proc/self/fd/`' and then the number of the descriptor you just got back from O_PATH, you can then pass it to `getxattr()` or `setxattr()` as a path, and it can’t be symlink-raced." He wasn't sure whether to attribute this code to Android or Red Hat developers, but a similar use of `/proc/self/fd/` can be found in the `open()` man page. 

Allison reiterated the main point of his talk: "The concept of pathnames is unusable on POSIX, completely. For a non-trivial application, for a regular person writing code on POSIX, you will have symlink races in your code."

Examples of (since fixed) CVEs were then provided, including one in the [Rust standard library](<https://blog.rust-lang.org/2022/01/20/cve-2022-21658.html>), which was [discussed extensively](<https://lwn.net/Articles/882070/>) here. In the last few minutes of the talk, Allison noted several proposed solutions offered by LWN readers, including [a special `prctl()` call](<https://lwn.net/Articles/882216/>) and [restrictions on when non-root symlinks are followed](<https://lwn.net/Articles/884413/>). He said that the [`MOUNT_NOSYMFOLLOW` mount option](<https://lwn.net/Articles/882487/>), which simply forbids following symlinks within a filesystem, is his preferred solution: "It’s perfect. It does exactly what we need." Allison's talk concluded on that point.

While it certainly seems desirable to forbid symlinks in the name of cleaning up the POSIX API, they are a frequently used system-administration tool. Several popular "symlink managers" exist. [Gnu Stow](<https://www.gnu.org/software/stow/>), for example, provides a way for administrators to install programs into a new directory hierarchy, such as `/usr/local/stow/packagename-version/`, and then create forests of symlinks from `/usr/local/bin/example` to `/usr/local/stow/packagename-version/bin/example`, using the minimum number of symlinks necessary. This makes it possible to "uninstall" a package simply by removing the symlinks with the help of `stow -D`. 

The `/etc/alternatives` system created by Debian allows administrators to switch between substitutable packages in a similar manner without forcing the uninstallation or reinstallation of either package. In a similar vein, the [Nix](<https://nixos.org/>) and [Guix](<https://guix.gnu.org/>) distributions make heavy use of symlinks -- a Guix profile consists of a tree of symlinks to packages installed within `/gnu/store/`, making it easy to switch between grouped combinations of specific versions of packages.

Banning symlinks entirely would break these use cases, but restricting their creation to the root user would most likely suffice. Users may still have other legitimate needs for symlinks, though, and substantially restricting them would likely be an unpopular change.

SambaXP has made the talk's [video](<https://www.youtube.com/watch?v=4JrY-DntoyU>) and [slides](<https://sambaxp.org/fileadmin/user_upload/sambaxp2022-Slides/Allison-Symlinks_considered_harmful.pdf>) available.
