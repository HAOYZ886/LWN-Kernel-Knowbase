---
title: Evolutionary development of a semantic patch using Coccinelle
url: https://lwn.net/Articles/380835/
date: "March 30, 2010"
category: "Development tools-Coccinelle"
author: "March 30, 2010 This article was contributed by Wolfram Sang"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

March 30, 2010

This article was contributed by Wolfram Sang

Creating patches is usually handwork; fixing one specific issue at a time. Once in a while though, there is janitorial work to be done or some infrastructure to change. Then, a larger number of issues have to be taken care of simultaneously, yet all of them are following the same basic pattern, e.g. a replacement. Such tasks are often addressed at the source-code level using scripts in sed, perl, and the like. This article examines the usage of [Coccinelle](<http://coccinelle.lip6.fr/>), a tool targeted at exactly those kinds of repetitive patching jobs. Because Coccinelle understands C syntax, though, it can handle those jobs much more easily. 

The major drawback of using scripts for code transformation is that they use non-trivial regular expressions in order to match previously unknown names, parse structures, and so forth. To simplify such tasks, "semantic patches"--patches that describe the _kinds_ of changes to be made, rather than the specific line and difference that come in normal patches--have been introduced along with Coccinelle to process them. Coccinelle translates the source files to an abstract representation, making it easier to deal with C expressions, isomorphisms, code paths and so on. For an introduction, refer to Valerie Aurora's [LWN article](<http://lwn.net/Articles/315686/>) or the Coccinelle web site. This article will provide a step-by-step description how a semantic patch came into existence once a certain problem was identified. 

Learning Coccinelle is still a bit challenging as information is scattered and, like with all languages, just listing the abilities is not even half of the story. Studying other semantic patches (in addition to asking on the mailing list) worked best for me, so in return this article describes the creation of a semantic patch from scratch. I would like to thank Julia Lawall for her immediate responses to my questions and bug reports. 

#### The problem

An issue was pointed out while developing an I2C driver for hardware monitoring: the driver serving an I2C slave device (called client) uses `i2c_set_clientdata()` to store a pointer to its private data structure, usually somewhere in the probe function. In the remove function, the driver was then supposed to clear the pointer to the data structure before freeing it, because clients are not really removed but are just unbound from the driver. To prevent a dangling pointer in the still existing client, a typical fix looks like: 

```
+   i2c_set_clientdata(client, NULL);
            /* clientdata pointed to data before */
            kfree(data);
```

As this dangling pointer looks quite easy to miss, checking all drivers is a job perfectly suited for Coccinelle. The goal is a patch series fixing this flaw in I2C drivers all over the kernel tree. While the [patch series](<http://thread.gmane.org/gmane.linux.drivers.i2c/5674>) Coccinelle successfully created will probably not be merged directly, it helped in finding [a more generic solution](<http://thread.gmane.org/gmane.linux.drivers.i2c/5674/focus=5729>). It was agreed that the i2c-core should clear the pointer to the private data structure as there is no guarantee for such pointers after the remove. A follow-up patch series will likely be based on the semantic patch presented below. In any case, the creation process will be useful for similar tasks in the future. 

The task can be further divided into two sub-problems: 

  1. Find relevant `kfree()` calls, which have the private data structure as an argument 
  2. Check if clientdata is `NULL` already 

If the latter is not the case, a fix is needed. For the following examples, Coccinelle 0.2.2 and a 2.6.34-rc1 kernel were used. Older kernels can also be used to get the idea, of course. 

#### Find relevant kfree() calls

A typical remove() routine for an I2C driver looks like this (from `[drivers/rtc/rtc-pcf8563.c](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=blob;f=drivers/rtc/rtc-pcf8563.c;h=65f346b2fbaebfcf73b58759e6622d6e9ababefd;hb=57d54889cd00db2752994b389ba714138652e60c>)`): 

```
static int pcf8563_remove(struct i2c_client *client)
        {
            struct pcf8563 *pcf8563 = i2c_get_clientdata(client);
    
            if (pcf8563->rtc)
                rtc_device_unregister(pcf8563->rtc);
    
            kfree(pcf8563);
    
            return 0;
        }
```

The pointer to the data structure of interest was obtained using `i2c_get_clientdata()`. When the structure itself gets freed, then a check for the call setting clientdata to `NULL` is needed. So this combination of `i2c_get_clientdata()` and `kfree()` is of interest, keeping in mind that the name of the pointer and its type can be anything. As Coccinelle parses the C source on an abstract level, this is easily possible using a few so-called metavariables in the header of our matching rule. Those can then carry the actual naming as used in the source file. Always remember that Coccinelle works on an abstract level. It is quite easy to forget as most of us are used to standard patches on source-code level. A first attempt of our semantic patch having one rule may look like this: 

```
@@
        // This is the rule header; metavariables must be declared here
        type T;
        identifier client, data;
        @@
            // The matching rule itself:
            // Catch the clientdata
            T data = i2c_get_clientdata(client);
    
            // then anything in between is allowed
            ...
    
            // prepend the fix if kfree() is found
        +	i2c_set_clientdata(client, NULL);
            kfree(data);
```

For the pcf8563 example above, this patch matches. That means, after the first line of the rule, the metavariable `T` will carry the type `struct pcf8563 *`, `data` will carry the identifier `pcf8563` and `client` will carry the identifier `client`. Later use of these metavariables will, of course, be accordingly replaced. So, `kfree(data)` will in fact look for `kfree(pcf8563)`. As this is also found, the match is complete and the line containing the fix will be added. 

But the patch did not find all relevant places. The `probe()` function also has a dangling pointer in the error path. It wasn't matched as it uses `i2c_set_clientdata()` instead of `i2c_get_clientdata()`. So there should be an alternation in the semantic patch handling both cases. And to make it short, a third variant is necessary because other drivers use `i2c_get_clientdata()` without declaring the type on the same line. It is usually a good idea to do a little bit of grepping first to get an idea in what ways functions are called. Here is the patch including all alternations marked by "`(`", "`|`", and "`)`" in the first column: 

```
@@
        type T;
        identifier client, data;
        @@
        // Check if function uses clientdata
        (
            i2c_set_clientdata(client, data);
        |
            data = i2c_get_clientdata(client);
        |
            T data = i2c_get_clientdata(client);
        )
            // anything in between is allowed
            ...
        +	i2c_set_clientdata(client, NULL);
            kfree(data);
```

Surprisingly, there is still no fixup for the `probe()` function. Why is that? The "..." operator in Coccinelle matches if and only if it matches for all code paths taken. This is to ensure consistency of the modifications. It usually makes a lot of sense, however, this case is an exception. As it is written now, the lower block of the patch says "anything in between is allowed, but then a `kfree(data)` must follow on all paths". Of course, the `probe()` routine does not free the structure if all went well because the driver is going to use it. So, the above rule will not match on this path and thus will fail entirely. What is needed here is a "may exist or may not exist" operator. This is, similar to regular expressions, "?". After changing the `kfree()` line to the following 

```
? 	kfree(data);
```

the meaning of the lower block changes to "anything in between is allowed and `kfree(data)` _may_ occur later". That implies that, if it occurs, the fix connected to `kfree(data)` will be applied as well, so finally there is the second match. 

#### Check if clientdata is freed already

When applying this semantic patch to the whole `rtc` subdirectory, there are a number of fixes, but also false positives, i.e. the pointer has correctly been cleared already by the driver, which is now done twice. To fix this, an alternation can be used again. Like in many languages, an alternation is short-cut if one condition is already met. So the replacing part can be done like this: 

```
(
            // If this pattern is found, clientdata is set to NULL before data is freed.
            // Do nothing and skip the rest of the alternation
            i2c_set_clientdata(client, NULL);
            ...
            kfree(data);
        |
            // Otherwise apply a fix if kfree() has been found in some code path
            // (doesn't need to be in all paths).
        +	i2c_set_clientdata(client, NULL);
        ? 	kfree(data);
        )
```

If the first block is met, the driver does the right thing. There still is a match, but no output is produced because no lines are added or removed. If this is not the case, the fix is applied (if needed). While being here, a few drivers clear the pointer after they free the structure. The other way around would be cleaner, so the following snippet is the third alternation: 

```
+	i2c_set_clientdata(client, NULL);
            kfree(data);
            ...
        -	i2c_set_clientdata(client, NULL);
```

The final version of the semantic patch is hopefully less frightening: 

```
@@
        type T;
        identifier client, data;
        @@
    
        // Check if function uses clientdata
        (
            i2c_set_clientdata(client, data);
        |
            data = i2c_get_clientdata(client);
        |
            T data = i2c_get_clientdata(client);
        )
            // Anything in between is OK
            ...
        (
            // If this pattern is found, clientdata is set to NULL before data is freed.
            // Do nothing and skip the rest of the alternation
            i2c_set_clientdata(client, NULL);
            ...
            kfree(data);
        |
            // If this pattern is found, clientdata is set to NULL after data is freed.
            // Move it to the front and skip the rest of the alternation
        +	i2c_set_clientdata(client, NULL);
            kfree(data);
            ...
        -	i2c_set_clientdata(client, NULL);
        |
            // Otherwise apply a fix if kfree() has been found in some code path
            // (doesn't need to be in all paths).
        +	i2c_set_clientdata(client, NULL);
        ? 	kfree(data);
        )
```

This matched 96 drivers in 23 directories, changing 213 lines. Note that one really should review those patches afterward. There might be issues which lead to further improvement of the semantic patch. Or there are problematic parts in the source code, but they need to be handled manually. For example, in this patch series, there was once a `kfree()` missing, so a memory leak was discovered. Also check the Coccinelle output for anomalies. In this case, there are some exceptions regarding "inconsistent control-flow paths". That means, the source code was modified in such a way that code paths outside our match would also be affected. An example is a simple error path in a probe function (excerpt from [drivers/gpio/pcf857x.c](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=blob;f=drivers/gpio/pcf857x.c;h=29f19ce3e80f192b071277a69bf1da9b67b8627f;hb=57d54889cd00db2752994b389ba714138652e60c>)): 

```
gpio = kzalloc(sizeof *gpio, GFP_KERNEL);
            if (!gpio)
                return -ENOMEM;
    
            ... /* set 'status' according to initialization */
    
            if (status < 0)
                goto fail;  /* clientdata not used yet! */
    
            ...
    
            i2c_set_clientdata(client, gpio);
    
            ...
    
            status = gpiochip_add(&gpio->chip);
            if (status < 0)
                goto fail;  /* clientdata was modified */
    
            ...
    
        fail:
            dev_dbg(...)
            /* 'i2c_set_clientdata(client, NULL)' placed here would be executed for all jumps to 'fail'! */
            kfree(gpio);
            return status;
```

As seen, a jump to `fail` can happen after or before clientdata was set to the private data structure. The latter case is outside the scope of the above semantic patch and would still modify its code path. In this example, the change is harmless as clientdata is still `NULL` and will be set to `NULL` again, but Coccinelle cannot know and outputs a warning. It is possible to enforce inconsistent changes using the command-line option `-allow_inconsistent_paths`, but it is marked as dangerous in the help text for a reason. Either triple-check the outcome or just handle the exceptions manually. 

#### Conclusion

The article is meant to incrementally describe the creation of a semantic patch using Coccinelle. While the result is working and the patch series was submitted, be aware that the semantic patch here is primarily meant for educational purposes; more advanced features available in Coccinelle have been left out. 

One has to get used to a slightly different way of thinking regarding patches along with learning some new syntax when getting started with Coccinelle. The intention of this article was to demonstrate that it is no major task, though. Once the basic stuff is familiar, semantic patches are easier to understand than scripts with loads of regular expressions. Coccinelle has also been around for some time now and produced a number of useful patch series (available via kernel-janitors), so it is not in alpha stage anymore. In the future, being able to read semantic patches will become increasingly important. Larger tasks, like API changes, might start being [done in an automatic fashion](<http://thread.gmane.org/gmane.linux.kernel/957411/focus=957657>). Coccinelle is a handy tool, and trying it out is likely to pay off.
