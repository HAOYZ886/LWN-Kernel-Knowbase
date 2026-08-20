---
title: A critical look at sysfs attribute values
url: https://lwn.net/Articles/378884/
date: "March 17, 2010"
category: "Development model-Code review"
author: "March 17, 2010 This article was contributed by Neil Brown"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

March 17, 2010

This article was contributed by Neil Brown

One of the many memorable lines from Douglas Adams's famous work _The Hitchhiker's Guide to the Galaxy_ was the accusation, probably leveled by supporters of the Encyclopedia Galactica, that the Hitchhiker's Guide was "unevenly edited" and "contains many passages which simply seemed to its editors like a good idea at the time." With small modifications, such as replacing "edited" with "reviewed", this description seems very relevant to the Linux kernel, and undoubtedly many other bodies of software, whether open or closed, free or proprietary. Review is at best "uneven". 

It isn't hard to find complaints that the code in the Linux kernel isn't being reviewed enough, or that we need more reviewers. The creation of tags like "Reviewed-by" for patches was in part an attempt to address this by giving more credit to reviewers and there by encouraging more people to get involved in that role. 

However one can equally well find complaints about too much review, where developers cannot make progress with some feature because, every time they post a revision, someone new complains about something else and so, in the pursuit of perfection, the good is lost. Similarly, though it does not seem to be a problem lately, there have been times when lots of review would simply result in complaints about white-space inconsistency and spelling mistakes -- things that are worth correcting, but not worth burying a valuable contribution under. 

Finding the right topic, the right level, and the right forum for review is not easy (and finding the time can be even harder). This article doesn't propose to address those questions directly, but rather to present a sample of review - a particular topic at a particular level on a particular forum, in the hope that it will be useful. The topic chosen, largely because it is something that your author has needed to work with lately without completely understanding, is "sysfs", the virtual filesystem that provides access to some of the internals of the Linux kernel. And in particular, the attribute files that expose the fine detail of that access. 

The level chosen is a high-level or holistic view, asking whether the implementation matches the goals, and at the same time asking whether the goals are appropriate. And the forum is clearly the present publication. 

#### Sysfs and attribute files

Sysfs has an interesting history and a number of design goals, both of which are worth understanding, but neither of which will be examined here except in as much as they reflect specifically the chosen topic: attribute files. The key design goal relating to attribute files is the stipulation - almost a mantra - of "one file, one value" or sometimes "one item per file". The idea here is that each attribute file should contain precisely one value. If multiple values are needed, then multiple files should be used. 

A significant part of the history behind this stipulation is the experience of "procfs" or `/proc`. `/proc` is a beautiful idea that unfortunately grew in an almost cancerous way to become widely despised. It is a virtual filesystem that originally had one directory for each process that was running, and that directory contained useful information about the running process in various files. 

There is clearly more that just processes that could usefully be put in a virtual filesystem, and, with no clear reason to the contrary, things started being added to procfs. With no real design or structure, more and more information was shoe-horned into procfs until it became an unorganised mess. Even inside the per-process directories procfs isn't a pretty sight. Some files (e.g. `limits`) contain tables with column headers, others (e.g. `mounts`) have tables without headers, and still others (e.g. `status`) have rows labeled rather than columns. Some files have single values (e.g. `wchan`) while others have lots of assorted and inconsistently formatted values (e.g. `mountstats`). 

Against this background of disorganisation and the attendant difficulty of adding new fields without breaking applications, sysfs was declared to have a new policy - one item per file. In fact, in his excellent (though now somewhat out-dated) [article on the Driver Model Core](<http://www.linuxjournal.com/article/6717>), Greg Kroah-Hartman even asserted that this rule was "enforced" (see the side bar on "sysfs"). 

It would not be fair to hold Greg accountable to what could have been a throw-away line from years ago, and I don't wish to do that. However that comment serves well in providing a starting point and a focus for reviewing the usage of attribute files in sysfs. We can ask if the rule really is being enforced, whether the rule is sufficient to avoid past mistakes, and whether the rule even makes sense in all cases. 

As you might guess the answers will be "no", "no" and "no", but the explanation is far more enlightening than the answer. 

#### Is it enforced?

The best way to test if the rule has been enforced is to survey the contents of sysfs - do files contain simple values, or something more? As a very rough assessment of the complexity of the contents of sysfs attribute file, we can issue a simple command: 

```
find /sys -mount -type f | xargs wc -w | grep -v ' total$'
```

to get a count of the number of words in each attribute file (the "-mount" is important if you have `/sys/kernel/debug` mounted, as reading things in there can cause problems). 

Processing these results from your author's (Linux 2.6.32) notebook shows that of the 9254 files, 1189 are empty and 7168 have only one word. It seems reasonable to assume these represent only one value (though many of the empty files are probably write-only and this mechanism gives no information about what value or values can be written). This leaves 897 (nearly 10%) which need further examination. They range from two words (487 cases) to 297 words (one case). 

While there are nearly 900 files, there are less than 100 base names. If we filter out some common patterns (e.g. `gpe` _%X_), the number of distinct attributes is closer to 62, which is a number that can reasonably be examined manually (with a little help from some scripting). Several of these multi-word attribute files contain non-ASCII data and so are almost certainly single values in some reasonable sense. Others contain strings for which a space is a legal character, such as "Dell Inc.", "i8042 KBD port" or "write back". So they clearly are not aberrations from the rule. 

There is a small class of files were the single item stored in the file is of an enumerated type. It is common for the file in these cases to contain all of the possible values listed which still seems to hold true to the "one item per file" rule. However there are three variations on this theme: 

  * In some cases, such as the "`queue/scheduler`" attribute of a block device, or the "`trigger`" attribute of an LED device, all of the possible options are listed, and the currently active one is enclosed in brackets, thus: 

```
noop anticipatory deadline [cfq]
```

  * In the second variation there are two files, one which contains the list of possibilities, as with "`cpufreq/scaling_available_governors`" and one which contains the currently-selected value, "`cpufreq/scaling_governor`". 

  * Finally, and this could be just a special case of one of the above, we have "`/sys/power/state`" for which there is no current value, so it just contains a list of the possible values. 

These are all examples of attribute files that do clearly contain just one value or item, but happen to use multiple words is various ways to describe those values. They are false-positives of our simplistic tool for finding complex attribute values. 

However there are other multi-word attribute files that are not so easily explained away. `/sys/class/bluetooth` contains some class attributes such as `rfcomm`, `l2cap` and `sco`. Each of these contains structured data, one record per line with 3 to 9 different datums per record (depending on the particular file), the first datum looking rather like the BD address of a local blue-tooth interface. 

This appears to be a clear violation of the "one item per file" policy. The files do appear to be very well structured and so easy to parse, so it is tempting to think that they should be safe enough. However sysfs attribute files are limited in size to one page - typically 4KB. If the number of entries in these files ever gets too large (about 70 lines in the `l2cap` file), accesses to the file will start corrupting memory, or crashing. Hopefully that will never happen, but "hope" is not normally an acceptable basis for good engineering. From a conversation with the bluetooth maintainer it appears that there are plans to move these files to "debugfs" where they can benefit from the "seq_file" implementation, also used widely in `/proc`, which allows arbitrarily large files. 

Some other examples include "`/sys/devices/system/node/node0/meminfo`" which appears to be a per-node version of "`/proc/meminfo`" and is clearly multiple values, and the "`options`" attributes in `/sys/devices/pnp*/*` which appear to contain exactly the sort of _ad hoc_ formatting of multiple values of multiple types that people find so unacceptable in `/proc`. The pnp "`resources`" files are similarly multi-valued, though to a lesser extent. 

As a final example of a lack of enforcement, the PCI device directory for the (Intel 3945) wireless network in this notebook contains a file called "`statistics`" which contains a hex dump of 240 bytes of data, complete with ASCII decoding at the end of each line such as: 

> 

```
>     02 00 03 00 d9 05 00 00 28 03 00 00 45 02 00 00  ........(...E...
>     0d 00 00 00 00 00 00 00 00 00 00 00 d6 00 00 00  ................
>     b1 02 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
>     00 00 00 00 00 00 00 00 67 00 00 00 00 00 00 00  ........g.......
>
```

This is surely not the sort of thing that sysfs was intended to report. If anything, this looks like it should be a binary attribute, not a doubly-encoded ASCII file. 

So to answer our opening question, "no", the one item per file rule is not enforced in any meaningful way. Certainly the vast majority of attribute files do contain just one item and that is good. But there are a number which contain multiple values in a variety of different ways. And this number is only likely to grow as people either copy the current bad examples, or find new use cases that don't seem to fit the existing patterns, so invent new approaches which don't take the holistic view into account. 

#### Is the rule sufficient?

Our next question to ask is whether the stated rule for sysfs attributes is sufficient to avoid an increasingly unorganised and _ad  hoc_ sysfs following the unfortunate path of procfs. We have already seen at least one case where it isn't. We do not have a standardised way of representing an enumerated type in a sysfs attribute, and so we have at least two implementations as already mentioned. There is at least one more implementation (exposed in the "`md/level`" attribute of md/raid devices) where just the current value is visible and the various options are not. Having a standard here would be good for consistency and encourage optimal functionality. But we have no standard. 

A similar issue arises with simple numerical values that represent measurable items such as storage size or time. It would be nice if these were reported using standard units, probably bytes and seconds. But we find that this is not the case. Amounts of storage are sometimes reported as bytes (`/sys/devices/system/memory/block_size_bytes`), sometimes as sectors (`/sys/class/block/*/size`), and sometimes as kilobytes (`block/_*_ /queue/read_ahead_kb`). 

As these particular examples show, one way to avoid ambiguity is to include the name of the units (_bytes_ or _kb_ here) as part of the attribute name, a practice known as Hungarian notation. However this is far from uniformly applied with the examples given above being more the exception than the rule. 

Measures of duration face the same problem. Many times that the kernel needs to know about are substantially less than one second. However rather than use the tried-and-true decimal point notation for sub-unit values, some attribute files report in milliseconds (`unload_heads` in libata devices), some in microseconds (`cpuide/state _*_ /time`), and some are even in seconds (`/sys/class/firmware/timeout`). As an extra confusion there are some (...`/bridge/hello_time`) which use a unit that varies depending on architecture, from centiseconds to mibiseconds (if that is a valid name for 1-1024th part of a second). It is probably fortunate that there is no metric/imperial difference in units for time else we would probably find both of those represented too. 

And then there are truth values: On, on, 1, Off, off, 0. 

So it would seem that the answer to our second question is "no" too, though it is harder to be positive about this as there is no clearly stated goal that we can measure against. If the goal is to have a high degree of uniformity in the representation of values in attributes, then we clearly don't meet that goal. 

#### Does the requirement always make sense?

So the guiding principle of one item per file is not uniformly enforced, and it isn't really enough to avoid needless inconsistencies, but were it to be uniformly applied, would it really give us what we want, or is it too simplistic or too vague to be useful as a strict rule? 

A good place to start exploring this question is the "`capabilities/key`" attribute of "input" devices. The content of this file is a bitmap listing which key-press events the input device can possibly generate. The bitmap is presented in hexadecimal with a space every 64 bits. Clearly this is a single value - a bitmap - but it is also an array of bits. Or maybe an array of "long"s. Does that make is multiple values in a single attribute? 

While that is a trivial example which we surely would all accept as being a single value despite being many bits long, it isn't hard to find examples that aren't quite as clear cut. Every block device has an attribute called "`inflight`" which contains two numbers, the number of read requests that are in-flight (have been submitted, but not yet completed) and the number of write requests that are in-flight. Is this a single array, like the bitmap, or two separate values? There would be little cost to have implemented "`inflight`" as two separate attributes thus clearly following the rule, but maybe there would be little value either. 

The "`cpufreq/stats/time_in_state`" attribute goes one step further. It contains pairs, one per line, of CPU frequencies (pleasingly in HZ) and the total time spent at that frequency (unfortunately in microseconds). This it is more of a dictionary than an array. On reflection, this is really the same as the previous two examples. For both "`key`" and "`inflight`" the key is an enumerated type that just happens to be mapped to a zero-based sequence of integers. So in each case we see a dictionary. In this last case the keys are explicit rather than implicit. 

If we contrast this last example with the "`statistics`" directory in any "net" device (`net/_*_ /statistics`) we see that it is quite possible to put individual statistics in individual files. Were these 23 different values put into one file, one per line with labels, it is unlikely that anyone would accept that there was just one item in that file. 

So the question here is: where do we draw the line? In each of these 4 cases (`capabilities/key`, `inflight`, `time_in_state`, `statistics`) we have a 'dictionary' mapping from an enumerated type to a scalar value. In the first case the scalar value is a truth value represented by a single bit, in the others the scalar is an integer. The size of the dictionary ranges from 2 to 23 to several hundred for "`capabilities/key`". Is it rational to draw a line based on the size of the dictionary, or on the size of the value? Or should it be left to the developer - a direction that usually produces disastrous results for uniformity. 

The implication of these explorations seems to be that we must allow structured data to be stored in attributes, as there is no clear line between structured and non-structured data. "One item per file" is a great heuristic that guides us well most of the time, but as we have seen there are numerous times where developers find that it is not suitable and so deviate from the rules with a disheartening lack of consistency. 

It could even be that the firmly stated rule has a negative effect here. Faced with a strong belief that a collection of numbers really forms a single attribute, and the strongly stated rule that multi-valued attributes are not allowed, the path of least resistance is often to quietly implement a multi-valued attribute without telling anyone. There is a reasonable chance that such code will not get reviewed until it is too late to make a change. This can lead multiple developers to solve the same problem in different ways, thus exacerbating a problem that the rule was intended to avoid. 

So to answer our third question, "no", the "one item per file" doesn't always make sense because it isn't always clear what "one item" is, and those places of uncertainty are holes for chaos to creep in to our kernel. 

#### Can we do better?

A review that finds problems without even suggesting a fix is a poor review indeed. The above identifies a number of problems, here we at least discuss solutions. 

The problem of existing attributes that are inappropriately complex or inconsistent in their formatting does not permit a quick fix. We cannot just change the format. At best we could provide new ways to access the same information, and then deprecate the old attributes. It is often stated that once something enters the kernel-userspace interface (which includes all of sysfs) it cannot be changed. However the existence of CONFIG_SYSFS_DEPRECATED_V2 disproves this claim. A policy that permits and supports deprecation and removal of sysfs attributes on an on-going basis may cause some pain but would be of long-term benefit to the kernel, especially if we expect our grandchildren to continue developing Linux. 

The problem that there is a clear need for structured data in sysfs attributes is probably best addressed by providing for it rather than ignoring or refuting it. Creating a format for representing arbitrarily structured data is not hard. Agreeing on one is much more of a challenge. XML has been enthusiastically suggested and vehemently opposed. Something more akin to the structure initialisations in C might be more pleasing to kernel developers (who already know C). 

Your author is currently pondering how best to communicate a list of "known bad blocks" on devices in a RAID between kernel and userspace. sysfs is the obvious place to manage the data, but one file per block would be silly, and a single file listing all bad blocks would hit the one-page maximum at about 300-400 entries, which is many fewer than we want to support. Having support for structured sysfs attributes would help a lot here. 

The final problem is how to enforce whatever rules we do come up with. Even with a very simple rule that is easily and often repeated and is heard by many, knowing the rule is not enough to cause people to follow the rule. This we have just seen. 

The implementation of sysfs attribute files allows each developer to provide an arbitrary text string which is then included in the sysfs file for them. This incredible flexibility is a great temptation to variety rather than uniformity. While it may not be possible to remove that implementation, it could be beneficial to make it a lot easier to build sysfs attributes of particular well supported types. For example duration, temperature, switch, enum, storage-size, brightness, dictionary etc. We already have a pattern for this in that module parameters are much easier to define when they are of a particular type - as can be seen when exploring `include/linux/moduleparam.h`. The moduleparam implementation focuses more on basic types such as int, short, long etc. For sysfs we are more interested in higher level types, however the concept is the same. 

If most of sysfs were converted over to using an interface that enforces standardised appearance, it would become fairly easy to find non-standard attributes and then either challenge them, or enhance the standard interface to support them. 

#### In Closing

It must be said that hindsight gives much clearer vision than foresight. It is easy to see these issues in retrospect, but would have been harder to be ready to guard against them from the start. While sysfs could possibly have had a better design, it could certainly have had a worse one. Creating imperfect solutions and then needing to fix them is an acknowledged part of the continuous development approach we use in the Linux kernel. 

For entirely internal subsystems, we can and do fix things regularly without any concern for legacy support. For external interfaces, fixing things isn't as easy. We need to either carry unsightly baggage around indefinitely or work to remove that which doesn't work, and encourage the creation only of that which does. Is it wrong to dream that our grandchild might work with a uniform and consistent `/sys` and maybe even a `/proc` which only contains processes?
