---
title: "A filesystem \"change journal\" and other topics"
url: https://lwn.net/Articles/755277/
date: "June 4, 2018"
category: Filesystems
author: "By Jake Edge June 4, 2018 LSFMM"
---

By **Jake Edge**  
June 4, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

At the 2017 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Amir Goldstein [presented](<https://lwn.net/Articles/718802/>) his work on adding a superblock watch mechanism to provide a scalable way to notify applications of changes in a filesystem. At the 2018 edition of LSFMM, he was back to discuss adding NTFS-like [change journals](<https://msdn.microsoft.com/en-us/library/windows/desktop/aa363798\(v=vs.85\).aspx>) to the kernel in support of backup solutions of various sorts. As a second topic for the session, he also wanted to discuss doing more performance-regression testing for filesystems. 

Goldstein said he is working on getting the superblock watch feature merged. It works well and is used in production by his employer, CTERA Networks, but there is a need to get information about filesystem changes even after a crash. Jan Kara suggested that what was wanted was an indication of which files had changed since the last time the filesystem changes were queried; Goldstein agreed. 

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2018/lsf-goldstein-sm.jpg) ](<https://lwn.net/Articles/755292/>)

NTFS has a change journal and he is working on something similar for his company's products that is based on overlayfs. Changes that are made to the filesystem go into the overlay. For his use case, he does not need high accuracy, false positives are not a serious problem; it is sufficient to simply know that something may have changed. His application will then scan the directory to determine what, if anything, has actually changed. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Kent Overstreet asked about using filesystem sequence numbers, but Goldstein said that not all filesystems have them. He is going to continue with his overlayfs-based plan, but would rather see a generic API that could be used by others. 

Dave Chinner wondered if the inode version (`i_version`) field could be used. Goldstein said that he wants to be able to query the filesystem to get all of the changes that have happened since a particular point in time. Josef Bacik said that Btrfs has that feature. Goldstein said that both Btrfs and dm-thin (thin provisioning) will provide a list of blocks that have changed. 

Simply scanning the inode array at mount time to find updates since the last query would be easy, but could take a long time, Ted Ts'o said. Chinner said the real underlying problem is missed notifications—filesystem changes that the application is not notified about. If that problem is solved, there is no need to scan after a crash. 

Goldstein is using a journal of the fsnotify event stream to reconstruct lost events in the event of a crash. But Chinner is worried about missing change notifications because the fsnotify event does not make it into the journal before the crash. Kara suggested a new fsnotify event that would indicate the intent to change; it would be journaled before the actual change. Since false positives are not a problem, if the actual change does not happen (and the fsnotify event is not actually generated), everything will still work. 

Kara said that FreeBSD has a facility that provides something similar to the NTFS change journal. The API for that is already established and might provide inspiration for the Linux API. Goldstein said that he already has a way to solve his immediate problem; he has lots of ideas for additional features if he gets the time to work on them. 

#### Filesystem performance regressions

Goldstein then shifted gears; he would like to see more filesystem performance-regression testing and wanted to discuss that. Bacik said that some performance tests have been merged into xfstests recently and asked for more. He has created a way to get [fio](<http://fio.readthedocs.io/en/latest/fio_doc.html>) data dumped in JSON that can be pulled into a SQLite database for doing comparisons. 

Overstreet suggested that those tests should be run automatically; if it has to be done manually, it won't happen. But Bacik said he has been focused on just getting it running; he wondered if it would be more valuable to run performance tests every time or only when the developer wants to look at performance numbers. 

Chinner said that the performance testing is really only meaningful for him when he is doing A/B testing. Otherwise, various runs of the test suite might have different debug settings (e.g. lockdep), so the results would not be comparable. In order for the runs to be meaningful, they have to be done in a controlled and consistent environment. 

Al Viro wondered how much variability was being seen between test runs. In his testing he has seen lots of variability, which makes it even harder to compare the results between different kernel versions. The allowable variability before flagging a regression is defined in the tests, Bacik said; it is around 2% or so. Right now, the output from every run is stored in a database, but it is fairly rudimentary, he said. 

Kara said that [MMTests](<https://github.com/gormanm/mmtests>) gathers similar kinds of data. He has found that averages are not particularly useful because the data is so noisy run to run, especially if the difference between the kernels is large. Average plus standard deviation is a reasonable starting point. He is not opposed to incorporating something simple into xfstests, but is concerned that more complex tests just make the run-to-run variability so high that it makes it hard to find where an observed regression is coming from. 

Bacik said that Facebook rebases its kernels yearly and he would like to have a simple test to be sure that the performance hasn't gone down radically. He wants discrete tests that won't show a lot of variability. But he is not trying to catch small performance losses with these tests. He said that the tests that are there now are "better than nothing" and that nothing is what was there before. He asked again for more tests. He also asked that Ts'o and Chinner run the performance tests for ext4 and XFS, as he is doing for Btrfs.
