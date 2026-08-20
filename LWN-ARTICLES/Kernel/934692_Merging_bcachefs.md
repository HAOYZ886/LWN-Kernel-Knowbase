---
title: Merging bcachefs
url: https://lwn.net/Articles/934692/
date: "June 16, 2023"
category: "Filesystems-bcachefs"
author: "By Jake Edge June 16, 2023 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
June 16, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The [bcachefs filesystem](<https://bcachefs.org/>), and the process for getting it upstream, were the topics of a session led remotely by Kent Overstreet, creator of bcachefs, at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>). He has also discussed bcachefs in previous editions of the summit, [first in 2018](<https://lwn.net/Articles/755276/>) and [at last year's event](<https://lwn.net/Articles/895266/>); in both of those cases, the question of getting bcachefs merged into the mainline kernel came up, but that merge has not happened yet. This time around, though, Overstreet seemed closer than ever to being ready to actually start that process. 

He began his talk by noting that he had been saying bcachefs is almost ready for merging for some time now; "now I'm saying, let's finally do it". He wanted to report on the status of the filesystem and on why it is ready now for upstreaming, but he wanted to use the bulk of the session to discuss the process of doing so. "It's a massive, 90,000-lines-of-code beast" that needs to get reviewed, so there is a need to figure out the process to do that review. 

His goal with bcachefs is to have the "performance, reliability, scalability, and robustness of XFS with modern features". That's a high bar, and one that bcachefs has not yet reached, but "I think we're pretty far along". People are running bcachefs on 100TB filesystems "without any issues or complaints"; he is waiting for the first 1PB filesystem. "Snapshots scale beautifully", which is not true for Btrfs, based on user complaints, he said. 

#### Status

In the last year, there has been a lot of scalability work done, much of which required deep rewrites, including for the allocator, which dates back to [bcache](<https://bcache.evilpiepirate.org/>). There is a new "no copy-on-write" (nocow) mode and snapshots have been implemented. People are using the snapshots to do backups of MySQL databases, he said, which is a test of the robustness of the feature. 

[ ![\[Kent Overstreet\]](https://static.lwn.net/images/2023/lsfmb-overstreet-sm.png) ](<https://lwn.net/Articles/934695/>)

[Erasure coding](<https://en.wikipedia.org/wiki/Erasure_code>) is the last really big feature that he would like to get into bcachefs before upstreaming it. But he thinks "it's time to draw a line in the sand", so that can wait for a bit. There is still a lot of work to do, but "the big feature work is lessening"; he will be able to work on being a maintainer without having to disappear for a month to work on something, as he did for snapshots, for example. 

The bcachefs team is growing; Brian Foster at Red Hat has been doing a lot of great work on bug fixes, Overstreet said. Eric Sandeen has helped in attracting interest in bcachefs at Red Hat as well. There is a bi-weekly call on bcachefs development. There is automated testing infrastructure that has been added and it is "making my life much easier", Overstreet said. The test system runs in about half an hour and includes multiple runs of fstests as well as the "huge test suite" for bcachefs. 

Rust is something that he has been evangelizing about to "anyone who will listen"; he thinks "writing code in C, when we finally have a better option available, is madness". He loves to write code, but not to debug it; writing in Rust "just means a lot less time debugging". He intends to slowly rewrite bcachefs in Rust, which will be a ten-plus-year project, but the use of Rust in bcachefs has already started. Some of the user-space tools have been rewritten in Rust and someone is looking at moving some of that work into the kernel. 

#### Upstreaming

That morning he had [posted](<https://lwn.net/ml/linux-fsdevel/20230509165657.1735798-1-kent.overstreet@linux.dev/>) 32 preliminary patches adding infrastructure that bcachefs will need; those patches were already being reviewed, he said. The rest is 90,000 lines of code in 2,500 patches that he did not post; he did include a link to his [Git repository](<https://evilpiepirate.org/git/bcachefs.git>), where those patches live in a [bcachefs-for-upstream branch](<https://evilpiepirate.org/git/bcachefs.git/log/?h=bcachefs-for-upstream>). He then opened up the floor to discuss how those patches would be reviewed and, eventually, merged. 

Josef Bacik said that he thinks the response will be much the same as last year; filesystem developers are "really excited" to see bcachefs get merged. He does not plan to review the implementation of the filesystem itself and suspects that is generally true. The people who are working on it will review it; "trust yourselves for that part". The "generic stuff is what we need to review", once that is done, the rest of the filesystem code can be merged as far as he is concerned. That is, of course, up to Linus Torvalds. 

Overstreet said that one of his questions is: "what do we take to Linus?" He has spent the last year on process and infrastructure, getting a team together, working with Red Hat, putting together an automated test suite, and so on. Mike Snitzer remotely pointed out that a patch set that had recently been rejected contained two enormous patches that were essentially impossible to review; he contrasted that with the 2,500 fine-grained patches that make up bcachefs, which is much easier to digest. 

While Snitzer is not sure that having everyone go through them one-by-one in review is the right approach, the obvious effort that went into that patch series makes it easier to trust the code and the process that went into developing it. "You've done the heavy lifting by doing all of that work to split up patches." Overstreet said that it was a lot of work to rebase nearly the entire history, but that it came in handy around six months ago when Red Hat noticed some big performance regressions. He was able to use that history to do automated bisection and got almost all of the performance back. 

Bacik said that Torvalds is the "maintainer" responsible for merging a new filesystem, so it will be up to him to decide if he is willing to pull the full history into the mainline. It would be Bacik's preference to do so, because the history is "super useful", but that is not something that the people in the room can decide. He suggested that the pull request be more of a question about whether the full history was acceptable and, if not, what would be. 

One concern is that once bcachefs gets merged, it will be difficult for anyone besides Overstreet to deal with the bug reports, Amir Goldstein said. It is important that it be explained in the pull request; "I want to merge this and I have a team that can support this". Getting more help was one of the criteria before upstreaming, Overstreet said. He knew that if it was a one-man show and he got deluged with bug reports, he would "go insane and run away to South America"; Foster has been "a huge help", which is one of the things that makes him feel comfortable about merging at this point. 

Paradoxically, the recent [push to remove some filesystems](<https://lwn.net/Articles/886708/>) (e.g. ReiserFS) from the kernel is actually going to make it easier to add new ones, Ted Ts'o said. He can remember Hans Reiser being enthusiastic about his new filesystem, with a team to support it, but that all fell into disrepair over the years. The kernel project now has a path for removing filesystems after a deprecation cycle. The idea that "accepting a filesystem isn't forever, makes it a whole lot easier" to merge new ones. 

He also suggested breaking up the patch series into smaller, more reviewable chunks that collect up a small number of related patches. That would make it easier for people to review, say, all of the lockdep patches in one chunk. It would mean relaxing the general guideline about not merging infrastructure until its first caller is merged, which he is in favor of; he would amend that guideline to allow merging when it includes a pointer to the Git tree of the first caller. 

Overstreet thinks that the preliminaries that he posted earlier that day will not be too controversial and other than perhaps one or two "will just sail through". He noted that Christoph Hellwig had objected to the [`vmalloc_exec()` patch](<https://lwn.net/ml/linux-fsdevel/20230509165657.1735798-8-kent.overstreet@linux.dev/>), though that functionality is needed for bcachefs, Overstreet said. Since the talk, Mike Rapoport has [proposed the JIT allocator](<https://lwn.net/Articles/933867/>), which would solve the underlying problem. 

A remote participant said that Foster's experience had shown that the code base is approachable; once bcachefs is available, interested developers will be able to come up to speed and start working on it with few difficulties. Christian Brauner asked that there be a clear delineation for who else could step in and merge patches if Overstreet is unavailable. Brauner noted that the NTFS/NTFS3 maintainer disappeared and, even though there were people who were contributing to the filesystem, it was not clear "who could route patches upstream". Overstreet said that he would trust Foster in that role if "he is willing to step up to that". 

Brauner said that he thinks bcachefs is in "excellent shape to be upstreamed", but he is concerned with the number of filesystems in the kernel; he is glad to see that there are efforts to remove some of them. Changes that impact all of the filesystems in the tree "get painful very very fast" and, in some cases, there is no one available to review the changes. He would like the acceptance process to be more conservative; accepting NTFS/NTFS3 was "a huge mistake", for example. Brauner said that none of that was directed at bcachefs, but was a more general concern; filesystem acceptance and deprecation was taken up in a lightning talk ([YouTube video](<https://www.youtube.com/watch?v=-fpgPrTnX2g>)) later that day. 

Darrick Wong said that he had already started doing what Ts'o suggested in his [patches](<https://lwn.net/ml/linux-fsdevel/20230526000020.GJ11620@frogsfrogsfrogs/>) for [XFS online repair](<https://lwn.net/Articles/934561/>). He has a collection of infrastructure patches that refer to callers that are coming soon; he has convinced Dave Chinner that there is value in reviewing the infrastructure pieces while also looking at the bigger picture of where it is all leading. That helps him because he can stop "rebasing things repeatedly and having to play code golf, like moving small helper functions up and down in the patch set". Putting all of that stuff in a separate set of infrastructure patches helped him, though it did cause some complaints from reviewers, but there is now some precedent for that approach, he said. 

Overstreet said that he is not particularly concerned about the 30 or so "relatively uncomplicated" infrastructure patches that he needs to land. He is going to wait for the Acked-by and Reviewed-by tags to come in, but if they do not, then he will use the suggested approach "as a Plan B". With that, the session came to a close.
