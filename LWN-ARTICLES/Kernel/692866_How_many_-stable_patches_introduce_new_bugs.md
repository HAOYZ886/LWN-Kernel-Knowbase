---
title: "How many -stable patches introduce new bugs?"
url: https://lwn.net/Articles/692866/
date: "June 28, 2016"
category: "Development model-Stable tree"
author: "By Jonathan Corbet June 28, 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 28, 2016

The -stable kernel release process faces a contradictory set of constraints. Developers naturally want to get as many fixes into -stable as possible but, at the same time, there is a strong desire to avoid introducing new regressions there. Each -stable release is, after all, intended to be more stable than its predecessor. At times there have been complaints that -stable is too accepting and too prone to regressions, but not many specifics. But, it turns out, this is an area where at least a little bit of objective research can be done. 

#### Worries about -stable regressions

Back in April, Sasha Levin [announced](<https://lwn.net/Articles/683544/>) the creation of a new extra-stable tree that would only accept security fixes. That proposal was controversial, and, after an initial set of releases, Sasha would appear to have stepped away from this project. While he was defending it, though, he claimed that some -stable patches introduce their own bugs, and offered a suggestion for doubtful developers: ""Take a look at how many commits in the stable tree have a 'Fixes:' tag that points to a commit that's also in the stable tree."" That is exactly what your editor set out to do. 

While kernel changelogs have a fairly well-defined structure, they are still free-form text, so investigating this area requires a certain amount of heuristics and regular-expression work, but it can be done. The first step is to look at the `Fixes:` tag as suggested by Sasha. Any kernel patch that fixes a bug introduced by another patch is meant to carry a tag like: 

```
Fixes: 76929ab51f0ee ("kselftests/ftrace: Add hist trigger testcases")
```

In the reality there is some variation in the format, the most common of which is putting the word "commit" before the ID. One would think that the -stable tree, which is supposed to contain (almost) exclusively fixes, would have a `Fixes:` tag on almost every commit. In truth, less than half of the commits there carry such tags. A few of those without tags are, in fact, straightforward reverts of buggy patches. Git adds a recognizable line to the changelog of reverts, so, unless the developer has significantly changed that line, it is easy to determine which patch is being "fixed" when a revert is done. 

Either way, though, the ID for the patch that introduced the bug is almost invariably the ID used in the mainline tree — _not_ the ID of the patch as it appears in the stable tree. Fortunately, stable-tree patches are required to carry a line like: 

```
commit d7591f0c41ce3e67600a982bab6989ef0f07b3ce upstream.
```

The format of that line tends to vary too, but, once that is coped with, it turns out that something around 99% of the changesets in the stable tree can be mapped to their mainline equivalent. Or, more to the point, the mapping can be done in the other direction, allowing `Fixes:` tags to be associated with commits in a specific -stable series. So, a when `Fixes:` line exists, one can, as a rule, fairly easily determine whether the patch fixes a bug introduced by another -stable patch. 

#### The results

The most recent long-term support kernel is 4.4, which has had 14 stable updates thus far. Those updates contained 1,712 changesets, 632 of which contained some form of `Fixes:` tag. Of those, it turns out that 39 were fixes for other patches that had already appeared in 4.4-stable. So just over 2% of the patches going into 4.4-stable have proved (so far) to contain bugs of their own requiring further fixes. 

For the curious, here's the full set: 

> 4.4-stable patches with bugs  
> ---  
> Introduced| Fixed  
> v4.4.1 | [`43a2ba8c1a003c82`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=43a2ba8c1a003c82d9f411af8ebcf0fe1e4cbe3e>) | v4.4.1 | [`0dec73176d5592ca`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=0dec73176d5592cac85c055bba90e22093e3b7e2>)  
> v4.4.1 | [`b5398ab9d4540c95`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=b5398ab9d4540c953f097b28f575486db2331f52>) | v4.4.2 | [`29a928ff8c1055ab`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=29a928ff8c1055ab34a0b40d285cbb2e4b0a4fd0>)  
> v4.4.1 | [`f5b62074b31a2844`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=f5b62074b31a2844579831499defca2cf2264df2>) | v4.4.3 | [`434e26d6f6a000b8`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=434e26d6f6a000b8585c0eb64764a55daff65d20>)  
> v4.4.1 | [`5e226f9689d90ad8`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=5e226f9689d90ad8ab21b4a969ae3058777f0aff>) | v4.4.4 | [`3ba9b9f2409168fb`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=3ba9b9f2409168fb50d0a0758b922508e7885f48>)  
> v4.4.1 | [`e924c60db1b4891e`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=e924c60db1b4891e45d15a33474ac5fab62cf029>) | v4.4.10 | [`a9bd748299179a8d`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=a9bd748299179a8d8f8fcd937c74ab321981ab4d>)  
> v4.4.1 | [`f50c2907a9b3dfc1`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=f50c2907a9b3dfc1ba840e6cc9884cf77d9e44cc>) | v4.4.1 | [`9497f702ab82314d`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=9497f702ab82314dffa457823be91783ca5a4531>)  
> v4.4.2 | [`d2081cfe624b5dec`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=d2081cfe624b5decaaf68088ca256ed1b140672c>) | v4.4.9 | [`9fed24fe30c1217c`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=9fed24fe30c1217c640d2b38403034c2c7fdce12>)  
> v4.4.2 | [`144b7ecc3bd6fdf7`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=144b7ecc3bd6fdf791d54def453d73168d99569f>) | v4.4.3 | [`a40efb855068a20c`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=a40efb855068a20cf769425a799642aa95c57635>)  
> v4.4.2 | [`c9b1074e18b607f5`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=c9b1074e18b607f54ac195959d5a4daacaa81325>) | v4.4.8 | [`4b59a38da5983852`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=4b59a38da5983852008270e81140f611df6f0bfd>)  
> v4.4.2 | [`f2e274ce8bfe8ab9`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=f2e274ce8bfe8ab928ab1968a82a3b7eec27a69b>) | v4.4.2 | [`6bb06a4fa1894533`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=6bb06a4fa1894533beaab22f0c657866b0e61b06>)  
> v4.4.2 | [`1489f5d951089deb`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=1489f5d951089debd2a533e3a109fba9b36e3461>) | v4.4.10 | [`a7fa0a478a625039`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=a7fa0a478a625039ef0852e5606d1248cba093e4>)  
> v4.4.3 | [`bbfe21c87bd0f529`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=bbfe21c87bd0f529d19f077051a52d779c785c6c>) | v4.4.12 | [`fa5613b1f39ec020`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=fa5613b1f39ec020671705453eeceaaf57459e49>)  
> v4.4.3 | [`152fb02241b60ffb`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=152fb02241b60ffb8d406b87c68d1908478a205f>) | v4.4.6 | [`f3c83858c6aee893`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=f3c83858c6aee893f8a3555aab3b72ae72f0f39f>)  
> v4.4.3 | [`726ecfc321994ec6`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=726ecfc321994ec6ab044c1e3e5886408de991ac>) | v4.4.10 | [`f6ff7398220d7fda`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=f6ff7398220d7fda0f4d02b9c9755406d8169bc2>)  
> v4.4.3 | [`f4595e0081495b67`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=f4595e0081495b677a98c780e9ec1ab68ce89488>) | v4.4.3 | [`55e0d9869f1d3a6b`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=55e0d9869f1d3a6bbd5d1e864c0e866fe1247f97>)  
> v4.4.4 | [`3824f7874a752196`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=3824f7874a7521961642287b4b93f14afbc9565a>) | v4.4.6 | [`78939530542f409e`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=78939530542f409efa9f6c2a34c8463ab745aa64>)  
> v4.4.4 | [`b36e52c44ce67288`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=b36e52c44ce6728824546d8b5f05b844cede96f1>) | v4.4.6 | [`6f0679556b563bcd`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=6f0679556b563bcd3d433d5781454123f1d134c5>)  
> v4.4.4 | [`a83b349814dee660`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=a83b349814dee660caff0a40a22ac2f166c94a8b>) | v4.4.6 | [`f8456804460f5c23`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=f8456804460f5c232f097e72051beea063f16074>)  
> v4.4.4 | [`7ed338d4a9f58d88`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=7ed338d4a9f58d88cd9fda055425bc6e0b8f3865>) | v4.4.4 | [`556dfd8dae7d66b3`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=556dfd8dae7d66b35121ffa8198465e55a34d64d>)  
> v4.4.4 | [`7c465723d0b6f262`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=7c465723d0b6f2621f6c712035b117d744a51a8b>) | v4.4.5 | [`c5cbbec54fe71c4d`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=c5cbbec54fe71c4de2d34f8c0ec8fbfdd7f17339>)  
> v4.4.4 | [`fc90441e728aa461`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=fc90441e728aa461a8ed1cfede08b0b9efef43fb>) | v4.4.5 | [`25e8618619a5a46a`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=25e8618619a5a46aae253e1cc68eeaa0d44d2f52>)  
> v4.4.4 | [`996c591227d988ed`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=996c591227d988ed82e76080ebf4ed0f1f33e0f1>) | v4.4.7 | [`dc1441612fdb4ca2`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=dc1441612fdb4ca221e3a4aa32e39e74d020e386>)  
> v4.4.5 | [`7adb5cc0f39be29c`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=7adb5cc0f39be29c16fae035d9e30e332095bbbc>) | v4.4.6 | [`b59ea3efba4889ec`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=b59ea3efba4889ece81056883193d78b51db11d1>)  
> v4.4.5 | [`e75c4b65150f0997`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=e75c4b65150f099789821cb5e3f0efc964c1db05>) | v4.4.6 | [`97142f3009557c27`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=97142f3009557c27091c694f927041f28ef81573>)  
> v4.4.7 | [`4c8fe4f52755d469`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=4c8fe4f52755d4690a745f4e56b543c51add86fe>) | v4.4.9 | [`5a58f809d731c23c`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=5a58f809d731c23c0b898d2021903db8dee4466f>)  
> v4.4.7 | [`b1999fa6e8145305`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=b1999fa6e8145305a6c8bda30ea20783717708e6>) | v4.4.9 | [`34af67eb941ae537`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=34af67eb941ae5371110c9adbd5392c7a3aa841e>)  
> v4.4.7 | [`c045105c641ccbeb`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=c045105c641ccbeb6e94e87980cc8db870aa3961>) | v4.4.7 | [`19e0783ae96837e3`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=19e0783ae96837e30e94acdb0cc4ae935338a969>)  
> v4.4.7 | [`dff87fa52ddf26df`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=dff87fa52ddf26df67526d303d08226e7168560b>) | v4.4.9 | [`5582eb00f5b23622`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=5582eb00f5b2362234cccf542232101db61ffc8b>)  
> v4.4.7 | [`8cbac3c4f74d92bf`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=8cbac3c4f74d92bf04645a613e061ab4f9baa866>) | v4.4.13 | [`a87f69dceff5c93a`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=a87f69dceff5c93a7d8f70f2cb255e1fcbda83bb>)  
> v4.4.7 | [`7f47aea487df2dc2`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=7f47aea487df2dc281c7f64ff7430aff3b260af0>) | v4.4.9 | [`9d58f322ee18ffac`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=9d58f322ee18ffaca1e0b67d90ab811ad75e62a6>)  
> v4.4.7 | [`a918d2bcea6aab6e`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=a918d2bcea6aab6e671bfb0901cbecc3cf68fca1>) | v4.4.7 | [`6677a2ab036f2813`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=6677a2ab036f28134b60ad4ed6fd2e72db579b8a>)  
> v4.4.7 | [`791b5b0d2d01542a`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=791b5b0d2d01542a87af4b5f8fb2504ce2d5b352>) | v4.4.9 | [`67fb098f6f23ebab`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=67fb098f6f23ebab7b47ae517c161032dc161cd9>)  
> v4.4.7 | [`5b5abb9b85e97630`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=5b5abb9b85e97630e07b2b6d33f4739a2eb4e872>) | v4.4.9 | [`54aeb5854ec03315`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=54aeb5854ec03315a721268b8c207fcdcd7f298f>)  
> v4.4.9 | [`9d58f322ee18ffac`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=9d58f322ee18ffaca1e0b67d90ab811ad75e62a6>) | v4.4.9 | [`be5cbaf31cd318f8`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=be5cbaf31cd318f8aaeeff901f6d27232dfa965f>)  
> v4.4.9 | [`19a4e46b4513bab7`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=19a4e46b4513bab7d6b368175be2e24ad4665e5a>) | v4.4.11 | [`9df2dc6cf4adb711`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=9df2dc6cf4adb711545f48001b34f35fd3bb79ef>)  
> v4.4.11 | [`1575c095e444c927`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=1575c095e444c927f0ebcdeb179c460c8c3b7f1f>) | v4.4.14 | [`f5f16bf66d7e07e5`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=f5f16bf66d7e07e5a04f07226caefeaf3136c83a>)  
> v4.4.12 | [`098942bcf4b1d057`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=098942bcf4b1d0573119150be8f68459d36ebde8>) | v4.4.14 | [`5e8b53a4db947494`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=5e8b53a4db947494f1d808469a411f7f2f8bb3ca>)  
> v4.4.14 | [`c9bc125c922e8550`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=c9bc125c922e855055cd08b2ec064180218be161>) | v4.4.14 | [`e9c74337a7c03d33`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=e9c74337a7c03d33f2afd5bb341cc20ad209698c>)  
> v4.4.14 | [`2066499780e1455c`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=2066499780e1455c43833b5b34858124047331ff>) | v4.4.14 | [`fe1e4026ce9f0365`](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=fe1e4026ce9f03653288c743218ed70ee0a2c4e0>)  
  
There are a couple of things worth noting in these results. One is that nine of the bugs introduced into 4.4-stable were fixed in the same -stable release — and [some](<https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit?id=0dec73176d5592cac85c055bba90e22093e3b7e2>) were arguably not bugs at all. So those problems almost certainly did not actually affect any -stable users; taking those out reduces the number of actual -stable regressions in 4.4 (so far) to 30\. On the other hand, 2/3 of the changes in 4.4-stable carry no `Fixes:` tag, but the bulk of them should still certainly be bug fixes. Some of them, undoubtedly, fix regressions that appeared in -stable, but, in the absence of somebody with the time, patience, and alcohol required to manually examine nearly 1,100 patches, there is no way to say for sure how many do. 

To get some sort of vague sense of the regression rate, one can start with the fact that the number found here constitutes a hard floor — the rate must be at least that high. If one makes the assumption that the regression rates in patches without `Fixes:` tags is no higher than those with the tags, a simple ratio gives the ceiling for the overall rate. For 4.4, that places the regression rate somewhere in the range 2.3-6.2%. Results from some of the other -stable trees are: 

> Series|  Patches| ` Fixes:` |  # fixed|  %regressions|   
> ---|---|---|---|---|---  
> 4.6| 314| 144 | 2| 0.6-1.4% | [Details](<https://lwn.net/Articles/692874/#4.6>)  
> 4.5| 973| 437 | 9| 0.9-2.1% | [Details](<https://lwn.net/Articles/692874/#4.5>)  
> 4.4| 1,712| 632 | 39| 2.3-6.2% | [Details](<https://lwn.net/Articles/692874/#4.4>)  
> 3.14| 4,779| 1,098 | 105| 2.2-9.6% | [Details](<https://lwn.net/Articles/692874/#3.14>)  
  
In the end, the results are clearly noisy. There _are_ regressions that appear in the -stable tree, and one can make some estimates as to just how many they are. There is no target regression rate for -stable (assuming that a target of zero is unrealistic), so whether the numbers shown above are acceptable or not is probably a matter of perspective — and whether one has been personally burned by a -stable regression or not. 

One conclusion that _can_ tentatively be drawn is that the regression rates for more recent kernels seem to be lower. Some portion of that reduction certainly comes from the youth of those kernels — there just hasn't been time to find all of the bugs yet. But it may also be that the efforts that have been made to reduce regressions in -stable (in particular, holding -stable patches until after they have appeared in a mainline -rc release) are having some effect. 

In the end, nobody wants to see regressions in the -stable trees. But tightening down on patch acceptance to the point that regressions no longer appear there will almost certainly result in buggier kernels overall, since many good fixes will not be accepted. As with many things in engineering, picking stable patches involves tradeoffs; hopefully the addition of some metrics can help the community to decide whether those tradeoffs are being made correctly. 

The code used to generate these results can be found as part of the gitdm collection of cheesy data-mining tools, located at git://git.lwn.net/gitdm.git.
