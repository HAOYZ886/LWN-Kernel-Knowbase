---
title: Best practices for a big patch series
url: https://lwn.net/Articles/585782/
date: "February 12, 2014"
category: "Development model-Patch management"
author: "February 12, 2014 This article was contributed by Wolfram Sang"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

February 12, 2014

This article was contributed by Wolfram Sang

The kernel development process features thousands of developers all working together without stepping on each other's toes — very often, anyway. The modularity of the kernel is one of the biggest reasons for the smoothness of the process; developers rarely find themselves trying to work on the same code at the same time. But there are always exceptions, one of which is the large, cross-subsystem patch series. Merging such a series does not have to be a recipe for trouble, especially if a few guidelines are followed; this article offers some suggestions in that direction. 

Changing the whole kernel tree using a pattern has become a lot easier in recent years. There is more processing power available, example scripts are out there, and tools like [Coccinelle](<http://lwn.net/Articles/380835/>) are especially targeted for such tasks. While this is great for wide-ranging work like API changes and bug fixes across all drivers, handling a patch series spanning across various subsystems can be a bit cumbersome. Dependencies and responsibilities need to be clear, the granularity (i.e. number of patches) needs to be proper, and relevant information needs to reach all people involved. If these conditions are not met, maintainers might miss important details which means more work for both the submitter and the maintainer. The best practices described below are intended to make submitting such a patch series smooth and to avoid this unnecessary work. 

#### Patch organization

The first question to answer is: in what form should your changes be posted? Here are the most commonly used choices along with examples of when they were used. There are no strict rules when to use which approach (and there can't be), so the examples hopefully give you an idea what issues to consider and what might be appropriate for your series. 

  1. **Changing the whole tree at once** : Having one patch changing files tree-wide in one go has the advantage of immediately changing the API (no transition time). Once applied, it is done, ready, and there should be no cruft left over. Because only one maintainer is needed to merge the huge patch, this person can easily handle any dependencies that might exist. The major drawback is a high risk of merge conflicts all over the tree because so many subsystems are touched. This approach was used for [renaming `INIT_COMPLETION()` to `reinit_completion()`](<http://thread.gmane.org/gmane.linux.kernel/1579901>). 

  2. **Grouping changes per file** : Having one patch for every modified file gives each subsystem maintainer freedom regarding when to apply the patches and how to handle merge conflicts because the patches do not cross subsystems. However, if there are dependencies, this can become a nightmare ("Shall I apply patches 32-53 to my tree now? Do I have to wait until 1-5 are applied? Who does that? Or is there a V2 of the series coming?"). Also, a huge number of patches pollutes the Git history. This choice was used for [removing `platform_driver_probe()`](<http://thread.gmane.org/gmane.linux.kernel/1575149>) from bus masters like I2C and SPI. It was chosen to provide a more fine-grained bisectability in case something went wrong. 

  3. **Grouping changes per subdirectory** : Having a patch per subdirectory somewhat resembles a patch per subsystem. This is a compromise of the former two options. Fewer patches to handle, but still each subsystem maintainer is responsible for applying and for conflict resolution. When the pinctrl core became able to select the default state for a group of pins, the explicit function call doing that in drivers was [removed](<http://thread.gmane.org/gmane.comp.video.dri.devel/93507>) in this fashion. In another example, a number of drivers did sanity checks of resources before passing them to `devm_ioremap_resource()`. Because the function does its own checks already, the drivers could be [cleaned up](<http://thread.gmane.org/gmane.comp.video.dri.devel/84891>) a little, one subdirectory at a time. Finally, the notorious [UAPI header file split](<https://lwn.net/Articles/507794/>) was also handled this way. 

  4. **Drop the series** : Finally, some tasks are just not suitable for mass conversion. One example is changing device drivers to use the managed resources API (`devm_*` and friends). There are surely some useful patterns to remove boilerplate code here. Still, not knowing hardware details may lead to [subtle errors](<http://article.gmane.org/gmane.linux.drivers.i2c/17223>). Those will probably be noticed for popular drivers, but may introduce regressions for less popular ones. So, those patches should be tested on real hardware before they are applied. If you really want to do a series like this as a service to the community, you should then ask for and collect `Tested-by` tags. Expect the patches to go in individually, not as a series. Patches that do not get properly tested may never be applied. 

Of course, the decision of which form to use should be driven by technical reasons only, patch count statistics, in particular, should not be a concern. As mentioned before, there are no hard rules, but you can assume that changing the whole tree at once is usually frowned upon unless the dependencies require it. Also, try to keep the number of patches low without sacrificing flexibility. That makes changes per subdirectory a good start if you are unsure. In any case, say in the cover letter what you think would be best. Be open for discussion because approaches do vary. For example, I would have preferred if the removal of `__dev*` attributes would have been one huge patch instead of 358 small ones. As a result, be prepared to convert your series from one form into another. 

Note: To automatically create commits per subdirectory with `git`, the following snippet can be used as a basis. It reads a commit message template specified by `$commit_msg_template` to create the commit descriptions. There, it replaces the string `SEDME` with the directory currently being processed. 

```
dirs=$(git status --porcelain --untracked-files=no $startdir | \
    	 dirname $(awk '/^ M / { print $2 }') | sort -u)
    
        for d in $dirs; do
            git add --update $d/*.[ch]
            sed "s|SEDME|${d//|/\|}|" $commit_msg_template | git commit --quiet -F -
        done
```

An example commit message template might look like:: 

```
SEDME: calling foo() in drivers is obsolete
    
        foo() is called by the core since commit 12345678 ("call foo() in core").
        No need for the driver to do it.
    
        Signed-off-by: Wolfram Sang <wsa@the-dreams.de>
```

#### The procedure

With any patch series, the good old "release early, release often" rule holds true. Let people know what you are up to. Set up a public repository, push your complete branch there, and update it regularly. If the series is not trivial, send an RFC to collect opinions. For an RFC, it may be appropriate to start by patching only one subsystem rather than the whole tree, or to use a whole-tree patch this one time in order to keep the mail count low. Always send a cover letter and describe your aim, dependencies, public repositories, and other relevant information. 

Ask Fengguang Wu to build your branch with his great [kbuild test service](<https://lists.01.org/mailman/listinfo/kbuild>). When all issues are resolved and there are no objections, send the whole series right away. Again, don't forget a proper cover letter. In case of per-file or per-directory patches, the subsystem maintainers will pick up the individual patches as they see fit. Be prepared for this process to take longer than one development cycle. In that case, rerun your pattern in the next development cycle and post an updated series. Keep at it until done. 

If it has been agreed to use the all-at-once approach, there may be a subsystem maintainer willing to pick up your series and take care of needed fixups during the merge window (or maybe you will be asked to do them). If there is no maintainer to pick your series but appropriate `Acked-by` tags were given, then (and only then) it is up to you to send a pull request to Linus. Shortly after the -rc1 release is a good time for this, though it is best to agree on this timing ahead of time. Make sure you have reapplied your pattern on the relevant -rc1 release so that the patches apply. Ask Stephen Rothwell to pull your branch into linux-next. If all went well, send out the pull request to Linus. 

#### Whom to send the patches to

When you send the series, use `git send-email`. The linux-kernel mailing list is usually the best `--to` recipient. Manually add people and lists to CC if they should be interested in the whole series. 

For other CCs, `get_maintainer.pl` from the kernel `scripts` directory is the tool to use. It supports custom settings via `.get_maintainer.conf`, which must be placed in the kernel top directory. The option `--no-rolestats` should be in that file; it suppresses the printing of information about why an email address was added. This extra output may confuse `git` and is also seen as noise on the mailing lists. The other default options are sane, but the usage of `--git-fallback` depends on the series you want to send. For per-file changes, it makes sense to activate this feature, because it adds people who actually worked on the modified files. For per-subsystem and whole-tree changes, `--no-git-fallback` (the default) makes sense, because those changes are mostly interesting for maintainers, so individual developers don't need to be on CC. If they are interested in the series, they will usually read the mailing list of the subsystem and notice your work there. 

There is one last tricky bit left: the cover letter. If it has too few CCs, people who receive individual patches might miss it; they are then left wondering what the patches are trying to accomplish. On the other hand, copying the cover letter to everybody who is also on CC of the patches will usually result in rejected emails, because the CC list becomes too large. The rule of thumb here is: add all mailing lists which get patches to the cover letter. Below is a script that does exactly that. It can be used as a `--cc-cmd` for `git send-email`. If it detects a cover letter, it runs `get_maintainer.pl` on all patches, collecting only mailing lists (`--no-m` option.) If it detects a patch, it simply executes `get_maintainer.pl`. 

```
#! /bin/bash
        #
        # cocci_cc - send cover letter to all mailing lists referenced in a patch series
        # intended to be used as 'git send-email --cc-cmd=cocci_cc ...'
        # done by Wolfram Sang in 2012-14, version 20140204 - WTFPLv2
    
        shopt -s extglob
        cd $(git rev-parse --show-toplevel) > /dev/null
    
        name=${1##*/}
        num=${name%%-*}
    
        if [ "$num" = "0000" ]; then
            dir=${1%/*}
            for f in $dir/!(0000*).patch; do
                scripts/get_maintainer.pl --no-m $f
            done | sort -u
        else
            scripts/get_maintainer.pl $1
        fi
```

#### Conclusion

Applying patterns to the kernel tree is surely a useful tool. As with any tool, knowledge when to use it and how to properly handle it needs to be developed. This article is hopefully a useful contribution in that direction. The author hopes to inspire other developers and is open for discussion.
