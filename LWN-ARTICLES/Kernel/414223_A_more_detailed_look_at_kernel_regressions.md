---
title: A more detailed look at kernel regressions
url: https://lwn.net/Articles/414223/
date: "November 10, 2010"
category: "Development model-Regressions"
author: "By Jake Edge November 10, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
November 10, 2010

The number of kernel regressions over time is one measure of the overall quality of the kernel. Over the last few years, Rafael Wysocki has taken on the task of tracking those regressions and regularly reporting on them to the linux-kernel mailing list. In addition, he has presented a "regressions report" at the last few Kernel Summits [[2010](<https://lwn.net/Articles/412746/>), [2009](<https://lwn.net/Articles/357480/>), and [2008](<https://lwn.net/Articles/298596/#regressions>)]. As part of his preparation for this year's talk, Wysocki wrote a paper, [_Tracking of Linux Kernel Regressions_ [PDF]](</images/pdf/kernel_regressions.pdf>), that digs in deeply and explains the process of Linux regression tracking, along with various trends in regressions over time. This article is an attempt to summarize that work. 

A regression is a user-visible change in the behavior of the kernel between two releases. A program that was working on one kernel version and then suddenly stops working on a newer version has detected a kernel regression. Regressions are probably the most annoying kind of bug that crops up in the kernel development process, as well as the one of the most visible. In addition, Linus Torvalds has decreed that regressions may not be intentionally introduced--to fix a perceived kernel shortcoming for example--and that fixing inadvertent regressions should be a high priority for the kernel developers. 

There is another good reason to concentrate on fixing any regressions: if you don't, you really have no assurance that the overall quality of the code is increasing, or at least staying the same. If things that are currently working continue to work in the future, there is a level of comfort that the bug situation is, at least, not getting worse. 

#### Regression tracking process

To that end, various efforts have been made to track kernel regressions, starting with Adrian Bunk in 2007 (around 2.6.20), through Michał Piotrowski, and then to Wysocki during the 2.6.23 development cycle. For several years, Wysocki handled the regression tracking himself, but it is now a three-person operation, with Maciej Rutecki turning email regression reports into kernel bugzilla entries, and Florian Mickler maintaining the regression entries: marking those that have been fixed, working with the reporters to determine which have been fixed, and so on. 

The kernel bugzilla is used to track the regression meta-information as well as the individual bugs. Each kernel release has a bugzilla entry that tracks all of the individual regressions that apply to it. So, [bug #16444](<https://bugzilla.kernel.org/show_bug.cgi?id=16444>) tracks the regressions reported against the 2.6.35 kernel release. Each individual regression is listed in the "Depends on" field in the meta-bug, so that a quick look will show all of the bugs, and which have been closed. 

There is another meta-bug, [bug #15790](<https://bugzilla.kernel.org/show_bug.cgi?id=15790>), that tracks all of the release-specific meta-bugs. So, that bug depends on #16444 for 2.6.35, as well as [#21782](<https://bugzilla.kernel.org/show_bug.cgi?id=21782>) for 2.6.36, [#15310](<https://bugzilla.kernel.org/show_bug.cgi?id=15310>) for 2.6.33, and so on. Those bugs are used by the scripts that Wysocki runs to generate the "list of known regressions" which gets posted to linux-kernel after each -rc release. 

Regressions are added to bugzilla one week after they are reported by email, if they haven't been fixed the interim. That's a change from earlier practices to save Rutecki's time as well as to reduce unhelpful noise. Bugzilla entries are linked to fixes as they become available. The bug state is changed to "resolved" once a patch is available and "closed" once Torvalds merges the fix into the mainline. 

Regressions for a particular kernel release are tracked through the following two development cycles. For example, when 2.6.36 was released, the tracking of 2.6.34 regressions ended. When 2.6.37-rc1 was released, that began the tracking for 2.6.36, and once 2.6.37 is released in early 2011, tracking of 2.6.35 regressions will cease. That doesn't mean that any remaining regressions have magically been fixed, of course, and they can still be tracked using the meta-bug associated with a release. 

#### Regression statistics

To look at the historical regression data, Wysocki compiled a table that listed the number of regressions reported for each of the last ten kernel releases as well as the number that are still pending (i.e. have not been closed). For the table, he has removed invalid and duplicate reports from those listed in bugzilla. It should also be noted that after 2.6.32, the methodology for adding new regressions changed such that those that were fixed in the first week after being reported were not added to bugzilla. That at least partially explains the drop in reports after 2.6.32. 

> Kernel | # reports | # pending  
> ---|---|---  
> 2.6.26 | 180 |  1  
> 2.6.27 |  144 |  4  
> 2.6.28 |  160 |  10  
> 2.6.29 |  136 |  12  
> 2.6.30 |  177 |  21  
> 2.6.31 |  146 |  20  
> 2.6.32 |  133 |  28  
> 2.6.33 |  116 |  18  
> 2.6.34 |  119 |  15  
> 2.6.35 |  63 |  28  
> Total |  1374 |  157  
> Reported and pending regressions  
  
The number of "pending" regressions reflects the bugs that have been fixed since the release, not just those that were fixed during the two-development-cycle tracking period. In order to look more closely at what happens during the tracking period, Wysocki provides another table. That table separates the two most important events during the tracking period, which are the releases of the subsequent kernel versions (i.e. for 2.6.N, the releases of N+1 and N+2). 

For example, once the 2.6.35 kernel was released, that ended the period where the development focus was on fixing regressions in 2.6.34. At that point, the merge window for 2.6.36 opened and developers switched their focus to adding new features for the next release. Furthermore, once 2.6.36 was released, regressions were no longer tracked at all for 2.6.34. That is reflected in the following table where the first "reports" and "pending" columns correspond to the N+1 kernel release, and the second to the N+2 release. 

> Kernel|  # reports (N+1)|  # pending (N+1)|  # reports (N+2)|  # pending (N+2)  
> ---|---|---|---|---  
> 2.6.30|  122|  36|  170|  45  
> 2.6.31|  89|  31|  145|  42  
> 2.6.32|  101|  36|  131|  45  
> 2.6.33|  74|  33|  114|  27  
> 2.6.34|  87|  31|  119|  21  
> 2.6.35| 61|  28|  |    
> Reported and pending regressions (separated by release)  
  
The table shows that the number of regressions still goes up fairly substantially after the release the next (N+1) kernel. This indicates that the -rc kernels may not be getting as much testing as the released kernel does. In addition, the pending kernel numbers are substantially higher for the N+2 kernel release, at least in the 2.6.30-32 timeframe. Had that trend continued, it could be argued that the kernel developers were paying less attention to regressions in a particular release once the next release was out. But the 2.6.33-34 numbers are fairly substantially down after the N+2 release, and Wysocki says that there are indications that 2.6.35 is continuing that trend. 

#### Reporting and fixing regressions

![\[Open regressions graph\]](https://static.lwn.net/images/2010/regression-closed.png)

We can look at the number of outstanding regressions over time in one of the graphs from Wysocki's paper. For each kernel release, there are generally two peaks that indicate where the number of open regressions is highest. These roughly correspond with the end of the merge window and the release date for the next kernel version. Once past those maximums, the graphs tend to level out. 

There are abrupt jumps in the number of regressions that are probably an artifact of how the reporting is done. Email reports are generally batched up, with multiple reports being added at roughly the same time. Maintenance on the bugs can happen in much the same way, which results in multiple regressions closed in a short period of time. That leads to a much more jagged graph, with sharper peaks. 

In the paper, Wysocki did some curve fitting for the the 2.6.33-34 releases that corresponded reasonably well with the observed data. He noted that the incomplete 2.6.35 curve was anomalous in that it didn't have a sharp maximum and seemed to plateau, rather than drop off. He attributes that to the shortened merge window for 2.6.37 along with the Kernel Summit and Linux Plumbers Conference impacting the testing and debugging of the current development kernels. Nevertheless, he used the same curve fitting equations on the 2.6.35 data to derive a "prediction" that it would end up with slightly more regressions than .33 and .34, but still less than 30. It will be interesting to see if that is borne out in practice. 

#### Regression lifetime

![\[Lifetime graph\]](https://static.lwn.net/images/2010/regression-lifetime.png)

The lifetime of regressions is another area that Wysocki addresses. One of his graphs is reproduced above and shows the cumulative number of regressions whose lifetime is less than the number of days on the x-axis. He separates the regressions into two sets, those from kernel 2.6.26-30 and from 2.6.30-35. In both cases, the curves follow that of radioactive decay, which allows for the derivation of the half-life for a set of kernel regressions: roughly 17 days. 

The graph for 2.6.30-35 is obviously lower than that of the earlier kernels, which Wysocki attributes to the change in methodology that occurred in the 2.6.32 timeframe. Because there are fewer short-lived (i.e. less than a week) regressions tracked, that will lead to a higher average regression lifetime. The average for the earlier kernels is 24.4 days, while the later kernels have an average of 32.3 days. Wysocki posits that the average really hasn't changed and that 24.5 days is a reasonable number to use as an average lifetime for regressions over the past two years or so. 

#### Regressions by subsystem

Certain kernel subsystems have been more prone to regressions than others over the last few releases, as is shown in a pair of tables from Wysocki's paper. He cautions that it is somewhat difficult to accurately place regressions into a particular category, as they may be incorrectly assigned in bugzilla. There are also murky boundaries between some of the categories, with power management (PM) being used as an example. Bugs that clearly fall into the PM core, or those that are PM-related but the root cause is unknown, get assigned to the PM category, while bugs in a driver's suspend/resume code get assigned to the category of the driver. Wysocki notes that these numbers should be used as a rough guide to where regressions are being found, rather than as an absolute and completely accurate measure. 

> Category|  2.6.32|  2.6.33|  2.6.34|  2.6.35|  Total  
> ---|---|---|---|---|---  
> DRI (Intel)|  20|  7|  10|  12|  49  
> x86|  9|  13|  21|  6|  49  
> Filesystems|  7|  12|  8|  8|  35  
> DRI (other)|  10|  7|  10|  5|  32  
> Network|  12|  8|  6|  4|  30  
> Wireless|  6|  6|  11|  4|  27  
> Sound|  8|  9|  4|  2|  23  
> ACPI|  7|  9|  3|  2|  21  
> SCSI & ATA|  4|  2|  2|  2|  10  
> MM|  2|  3|  4|  0|  9  
> PCI|  3|  4|  1|  1|  9  
> Block|  2|  1|  3|  2|  8  
> USB|  3|  0|  0|  3|  6  
> PM|  4|  2|  0|  0|  6  
> Video4Linux|  1|  3|  1|  0|  5  
> Other|  35|  30|  35|  12|  112  
> Reported regressions by category  
  
The Intel DRI driver and x86 categories are by far the largest source of regressions, but there are a number of possible reasons for that. The Intel PC ecosystem is both complex, with many different variations of hardware, and well-tested because there are so many of those systems in use. Other architectures may not be getting the same level of testing, especially during the -rc phase. 

It is also clear from the table that those subsystems that are "closer" to the hardware tend to have more regressions. The eight rows with 20 or more total regressions--excepting filesystems and networking to some extent--are all closely tied to hardware. Those kinds of regressions tend to be easier to spot because they cause the hardware to fail, unlike regressions in the scheduler or memory management code, for example, which are often more subtle. 

> Category|  2.6.32|  2.6.33|  2.6.34|  2.6.35|  Total  
> ---|---|---|---|---|---  
> DRI (Intel)| 1| 2| 2| 5| 10  
> x86| 2| 2| 3| 2| 9  
> DRI (other)| 1| 3| 2| 3| 9  
> Sound| 5| 2| 0| 1| 8  
> Network| 2| 2| 1| 2| 7  
> Wireless| 1| 1| 1| 2| 5  
> PM| 4| 1| 0| 0| 5  
> Filesystems| 0| 0| 0| 5| 5  
> Video4Linux| 1| 3| 0| 0| 4  
> SCSI + SATA| 2| 0| 1| 0| 3  
> MM| 1| 0| 1| 0| 2  
> Other| 8| 2| 4| 8| 22  
> Pending regressions by category  
  
It is also instructive to look at the remaining pending regressions by category. In the table above, we can see that most of the regressions identified have been fixed, with only relatively few persisting. Those are likely to be bugs that are difficult to reproduce, and thus track down. Some categories, like ACPI, fall completely out of the table, which indicates that those developers have been very good at finding and fixing regressions in that subsystem. 

#### Conclusion

Regression tracking is important so that kernel developers are able to focus their bug fixing efforts during each development cycle. But looking at the bigger picture--how the number and types of regressions change--is also needed. Given the nature of kernel development, it is impossible to draw any conclusions from the data collected for any single release. By aggregating data over multiple development cycles, any oddities specific to a particular cycle are smoothed out, which allows for trends to be spotted. 

Since regressions are a key indicator of kernel quality, and easier to track than many others, they serve a key role in keeping Torvalds and other kernel developers aware of kernel quality issues. As the developers get more familiar with the "normal" regression patterns, it will become more obvious that a given release is falling outside of those patterns, which may mean that it needs more attention--or that something has changed in the development process. In any case, there is clearly value in the statistics, and that value is likely to grow over time.
