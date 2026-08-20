---
title: PostgreSQL considers seccomp() filters
url: https://lwn.net/Articles/800916/
date: "October 1, 2019"
category: "Security-seccomp"
author: "By Jake Edge October 1, 2019"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
October 1, 2019

A discussion on the [pgsql-hackers mailing list](<https://www.postgresql.org/list/pgsql-hackers/>) at the end of August is another reminder that the suitability of [`seccomp()`](<http://man7.org/linux/man-pages/man2/seccomp.2.html>) filters is likely more narrow than was hoped. Applying filters to the PostgreSQL database is difficult for a number of reasons and the benefit for the project and its users is not entirely clear. The discussion highlights the tradeoffs inherent in adding system-call filtering to a complex software suite; it may help crystallize the thinking of other projects that are also looking at supporting `seccomp()` filters. 

Joe Conway raised the idea in an [RFC patch posting](<https://lwn.net/Articles/800922/>). It added a way to filter system calls in the main postmaster process and, with a separate system-call list, in the per-session backends. It also showed how to generate the list of system calls that are being used by PostgreSQL under various workloads, such as the test targets in the Makefile or by running a specific application. Information on the system calls made is logged by the audit subsystem; those logs are then processed to produce the list. Once there is confidence that the list is complete—which may be a sticking point—the remaining system calls could be blocked so that executing them would cause an error. 

But Peter Eisentraut was [concerned](<https://lwn.net/Articles/800944/>) that the list is going to be incomplete due to the ""fantastic test coverage"" needed to generate it and that it will require constant maintenance to keep up with GNU C Library (glibc) and other changes. Beyond that, PostgreSQL extensions will need their own lists of allowed system calls. Conway [seems to see](<https://lwn.net/Articles/800947/>) the support as something that those interested will maintain for themselves, rather than having a list that the project will distribute. ""Perhaps most people never use this, but when needed (and increasingly will be required) it is available."" 

Tom Lane [suggested](<https://lwn.net/Articles/800948/>) that it made more sense to use some kind of static analysis to determine the system calls that PostgreSQL legitimately makes, rather than simply testing to produce the list. But he also doesn't quite see what threat model the feature is protecting against. Since it is PostgreSQL itself that is maintaining and configuring the system-call filter list, a compromise that allowed privileged code execution in PostgreSQL could just disable the filtering and restart PostgreSQL, making the filters moot: 

Given that we'll allow any syscall that an unmodified PG executable might use, it seems like the only scenarios being protected against involve someone having already compromised the server enough to have arbitrary code execution. OK, fine, but then why wouldn't the attacker just bypass libseccomp? Or tell it to let through the syscall he wants to use? Having the list of allowed syscalls be determined inside the process seems like fundamentally the wrong implementation. 

Joshua Brindle [thought](<https://lwn.net/Articles/800953/>) that at least blacklisting some high-risk system calls would help bolster the security of a system running PostgreSQL. Systemd has some predefined lists that might be used as a starting point. He is also concerned that since the feature is just one component of a full solution, looking at it in isolation is not the right approach: 

The goal is to prevent an ACE [arbitrary code execution] hole in Postgres from becoming a complete system compromise. This may not do it alone, and security conscious integrators will want to, for example, add seccomp filters to systemd to prevent superuser from disabling them. The postmaster and per-role lists can further reduce the available syscalls based on the exact extensions and PLs being used. Each step reduced the surface more and throwing it all out because one step can go rogue is unsatisfying. 

The fragility of `seccomp()` filters is also part of what [concerned](<https://lwn.net/Articles/800954/>) Andres Freund. He noted that there have already been PostgreSQL bug reports about `seccomp()` because of how it is used by some container-management systems. The system-call landscape is constantly shifting as well, he said, pointing to an [LWN article](<https://lwn.net/Articles/795128/>) about one `seccomp()`-related problem: 

There's regularly new syscalls (e.g. epoll_create1(), and we'll soon get openat2()), different versions of glibc use different syscalls (e.g. switching from open() to always using openat()), the system configuration influences which syscalls are being used (e.g. using vsyscalls only being used for certain clock sources), and kernel bugfixes change the exact set of syscalls being used. 

Lane [wondered](<https://lwn.net/Articles/800955/>) why SELinux was not being used; Brindle [made it clear](<https://lwn.net/Articles/800956/>) that while SELinux cannot be used to do system-call filtering, it is part of the overall PostgreSQL hardening effort. As an example of the kind of system call that could be blacklisted for PostgreSQL using `seccomp()`, Brindle pointed to [`madvise()`](<http://man7.org/linux/man-pages/man2/madvise.2.html>), which he said is not preventable by SELinux, not used by PostgreSQL, and ""a clear win in the dont-let-PG-be-a-vector-for-kernel-compromise arena"". 

But Freund [cautioned](<https://lwn.net/Articles/800957/>) that `madvise()` _is_ used by glibc as part of its `malloc()` implementation. ""That's *precisely* my problem with this approach."" As Lane [pointed out](<https://lwn.net/Articles/800958/>), calls that are buried deeply in dependencies of PostgreSQL are not going to be found easily via testing: 

So this makes a perfect example for [Peter Eisentraut's] point that testing is going to be a very fallible way of finding the set of syscalls that need to be allowed. Even if we had 100.00% code coverage of PG proper, we would not necessarily find calls like this. 

Yet another instance of `seccomp()` fragility was [raised](<https://lwn.net/Articles/800977/>) by Thomas Munro. The [`sync_file_range()`](<http://man7.org/linux/man-pages/man2/sync_file_range.2.html>) system call on PowerPC and Arm has a `sync_file_range2()` variant with better argument ordering; glibc helpfully remaps calls to that variant on the relevant architectures. But Docker and other container managers did not include `sync_file_range2()` in their whitelists, leading to unexpected errors. 

While generally acknowledging the problems mentioned, Brindle and Conway still think it makes sense to provide the hooks for the feature for those who want it. Brindle [said](<https://lwn.net/Articles/800979/>): 

The feature allows end users to define some sandboxing within PG. Nothing is being forced on anyone but we would like the capability to harden a PG installation for many reasons already stated. This is being done in places all across the Linux ecosystem and is IMO a very useful mitigation. 

But even for an optional feature, there is still a cost to PostgreSQL, Eisentraut [said](<https://lwn.net/Articles/800980/>): 

Features come with a maintenance cost. If we ship it, then people are going to try it out. Then weird things will happen. They will report mysterious bugs. They will complain to their colleagues. It all comes with a cost. 

Conway [noted](<https://lwn.net/Articles/800982/>) that PostgreSQL is already being run under `seccomp()` filters, however. He and Brindle think that it would be better for the project to proactively implement support: 

It is our assessment that PostgreSQL will be subject to seccomp willingly or not (e.g., via docker, systemd, etc.) and the community might be better served to get out in front and have first class support. 

Conway wondered if just adding the hooks to load the filters would be a path forward, but Lane [was not in favor](<https://lwn.net/Articles/800983/>) of putting the filter controls inside the PostgreSQL process. That ""seems like a fundamentally incorrect architecture"", he said. In order to be a ""credible security improvement"", the filters need to be imposed on the PostgreSQL processes from the outside. As might be guessed, Brindle and Conway disagreed with that characterization. Their company (Crunchy Data) has customers that need the feature, so they will continue to pursue it, Conway said. For the PostgreSQL mainline, though, it would seem that the feature is not really welcome—at least in its present form. 

While the appeal of filtering at the system-call level is strong, it is not entirely clear that it is the best way forward for everything. Processes like a browser's rendering engine, which was an initial `seccomp()` filtering target, are well suited to the approach. By their very nature, database engines—effectively general-purpose programming languages—do not really fit that mold. Adding system-call filters to something like Python (or Perl, Ruby, PHP, ...) is similarly problematic. Alternatives, at least for Linux, are not readily available, however, which may be causing people to try to fit round pegs in square holes.
