---
title: Semantic patching with Coccinelle
url: https://lwn.net/Articles/315686/
date: "January 20, 2009"
category: "Development tools-Coccinelle"
author: "January 20, 2009 This article was contributed by Valerie Aurora"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

January 20, 2009

This article was contributed by Valerie Aurora

We've all been there: You're tracking down some evil bug, and you have the sudden chilling realization that you're going to have to refactor an enormous chunk of code to fix it. You break out in a cold sweat as you run a quick grep over the source base: hundreds of lines of code to change! And the change is too complex to do with a script because it depends on the calling context, or requires adding a new variable to every caller. 

This happened to me last month when I was adding support for 64-bit file systems to e2fsprogs. I thought I was nearly finished when I discovered I needed to write (yet another) new interface and convert (yet another) several hundred lines of code to it. The changes were complex enough that I couldn't use a script, and simple enough that I wanted to claw my eyes out with the soul-killing boredom of doing it by hand. That's when the maintainer, Theodore Ts'o, suggested I look at [Coccinelle](<http://www.emn.fr/x-info/coccinelle/>) (a.k.a., spatch). 

### Coccinelle

Coccinelle is a tool to automatically analyze and rewrite C code. Coccinelle (pronounced cock'-see-nel) means "ladybug" in French, a name chosen because ladybugs eat other bugs. Coccinelle is not just another scripting language; it is aware of the structure of the C language and can make much more complex changes than are possible with pure string processing. For example, Coccinelle can make a particular change only in functions which are assigned to a function pointer in a particular type of array -- say, the `create` member of `struct inode_operations`. 

The input to the tool is the file(s) to be changed and a "semantic patch," written in [SmPL (Semantic Patch Language)](<http://www.emn.fr/x-info/coccinelle/docs/index.html>). SmPL looks a like a unified diff (a patch) with some C-like declarations mixed in. Here's an example: 

```
@@
        expression E;
        identifier fld;
        @@
    
        - !E && !E->fld
        + !E || !E->fld
```

This semantic patch fixes the bug in which the pointer is tested for NULL -- and then dereferenced if the pointer is NULL. An example of a bug this semantic patch found in the Linux kernel (and automatically generated the fix for): 

```
--- a/drivers/pci/hotplug/cpqphp_ctrl.c
        +++ b/drivers/pci/hotplug/cpqphp_ctrl.c
        @@ -1139,7 +1139,7 @@ static u8 set_controller_speed(struct controller
        *ctrl, u8 adapter_speed, u8 hp_
                for(slot = ctrl->slot; slot; slot = slot->next) {
                        if (slot->device == (hp_slot + ctrl->slot_device_offset))
                                continue;
        -               if (!slot->hotplug_slot && !slot->hotplug_slot->info)
        +               if (!slot->hotplug_slot || !slot->hotplug_slot->info)
                                continue;
                        if (slot->hotplug_slot->info->adapter_status == 0)
                                continue;
```

(More on the semantic patch format later.) 

Coccinelle is designed, written, and maintained by [Julia Lawall](<http://www.diku.dk/~julia>) at the [Department of Computer Science at University of Copenhagen](<http://www.diku.dk/english/>), [Gilles Muller](<http://www.emn.fr/x-info/gmuller/>) and [Yoann Padioleau](<http://aryx.kicks-ass.org/~pad/homepage.php>) at the Ecole des Mines de Nantes, and Rene Rydhof Hansen at the [Department of Computer Science of Aalborg University](<http://www.cs.aau.dk/en/welcome/>). Coccinelle is licensed under the GPL, however, it is written in OCaml, so the potential developer base is somewhat limited. 

The original goal of Coccinelle was to automate as much as possible the task of keeping device drivers up to date with the latest kernel interfaces. But the end result can do far more than that, including finding and fixing bugs and coding style irregularities. Over 180 patches created using Coccinelle have been [accepted into the Linux kernel](<http://www.emn.fr/x-info/coccinelle/#impact>) to date. 

### Coccinelle quickstart

Like many languages, SmPL is best learned through example. We'll run through one simple example here just to get started. After that, the Coccinelle web page has some [documentation](<http://www.emn.fr/x-info/coccinelle/#documentation>) and a plethora of [examples](<http://www.emn.fr/x-info/coccinelle/#examples>). 

First, [download Coccinelle](<http://www.emn.fr/x-info/coccinelle/#download>) and install it. I used the source version rather than any of the precompiled options. The Coccinelle binary is called `spatch`. 

As our example, say we have program with a lot of calls to `alloca()` that we would like to replace with `malloc()`. `alloca()` allocates space on the stack and can be more efficient and convenient than `malloc()`, but it is also compiler-dependent, non-standard, easy to use incorrectly, and has undefined behavior on failure. (Replacing `alloca()` with `malloc()` isn't enough, we also have to check the return value -- but that will come later.) 

Here is the C file we are working on: 

```
#include <alloca.h>
    
        int
        main(int argc, char *argv[])
        {
                unsigned int bytes = 1024 * 1024;
                char *buf;
    
                /* allocate memory */
                buf = alloca(bytes);
    
                return 0;
        }
```

We could make the replacement using a scripting language like `sed`: 

```
$ sed -i 's/alloca/malloc/g' test.c
```

But this will replace the string "alloca" anywhere it appears. The resulting diff: 

```
--- test.c
        +++ /tmp/test.c
        @@ -1,4 +1,4 @@
        -#include <alloca.h>
        +#include <malloc.h>
     
         int
         main(int argc, char *argv[])
        @@ -6,8 +6,8 @@
                 unsigned int bytes = 1024 * 1024;
                 char *buf;
     
        -        /* allocate memory */
        -        buf = alloca(bytes);
        +        /* mallocte memory */
        +        buf = malloc(bytes);
     
                 return 0;
         }
```

We can tweak our script to handle 90% of the cases: 

```
$ sed -i 's/alloca(/malloc(/g' test.c
```

But this script doesn't handle the case where a second function name has the first as a suffix, it depends on a particular coding style in which no white space comes between the function name and the open parenthesis, etc., etc. By now our simple `sed` script is a hundred-character monster. It can be done, but it's a pain. 

In Coccinelle, we'd use the following semantic patch: 

```
@@ expression E; @@
    
        -alloca(E)
        +malloc(E)
```

Put the C file in `test.c` and the above semantic patch in `test.cocci` and run it like so: 

```
$ spatch -sp_file test.cocci test.c
```

It should produce the following diff: 

```
--- test.c
        +++ /tmp/cocci-output-17416-b5450d-test.c
        @@ -7,7 +7,7 @@ main(int argc, char *argv[])
                 char *buf;
     
                 /* allocate memory */
        -        buf = alloca(bytes);
        +        buf = malloc(bytes);
     
                 return 0;
         }
```

Let's look at the semantic patch line by line. 

```
@@ expression E; @@
```

This declares the "metavariable" E as a variable that can match any expression -- e.g., `1 + 2`, `sizeof(x)`, `strlen(name) + sizeof(x) * 72`. When spatch processes the input, it sets the value of E to the argument to `alloca()`. The "`@@ @@`" syntax is chosen to resemble the line in a unified diff describing the lines to be patched. I don't find the resemblance particularly helpful, but the intention is well-taken. 

```
-alloca(E)
```

This line says to remove any call to the function `alloca()`, and to save its argument in the metavariable E for later use. 

```
+malloc(E)
```

And this line says to replace the call to `alloca()` with a call to `malloc()` and use the value of metavariable E as its argument. 

Now, we also want to check the return value of `malloc()` and return an error if it failed. We can do that too: 

```
@@
        expression E;
        identifier ptr;
        @@
    
        -ptr = alloca(E);
        +ptr = malloc(E);
        +if (ptr == NULL)
        +        return 1;
```

The resulting diff: 

```
--- test.c
        +++ /tmp/cocci-output-17494-22a573-test.c
        @@ -7,7 +7,8 @@ main(int argc, char *argv[])
                 char *buf;
     
                 /* allocate memory */
        -        buf = alloca(bytes);
        +        buf = malloc(bytes);
        +        if (buf == NULL)
        +                return 1;
     
                 return 0;
     }
```

Semantic patches can be far more complex. One of my favorite examples is the move of reference counting of the `Scsi_Host` structure out of drivers. Changing this required adding an argument to the function signature and removing a declaration and several other lines from each SCSI driver's `proc_info` function. The semantic patch, explained in detail in their OLS 2007 slides [ [PPT]](<http://www.emn.fr/x-info/coccinelle/semantic-patches-talk-ols07.ppt>) [[ODP]](<http://www.emn.fr/x-info/coccinelle/semantic-patches-talk-ols07.odp>), does all of this automatically. I recommend reading and re-reading this example until it sinks in. 

### Experience

My [first experience](<http://valhenson.livejournal.com/30409.html>) with Coccinelle was mixed. In theory, Coccinelle does exactly what I want -- automate complex changes to code -- but in practice the implementation is beta quality. I successfully used Coccinelle to make hundreds of lines of changes with less than a hundred lines of semantic patches, but only after working directly with the developers to get bug fixes and help figuring out SmPL features. Coccinelle is one of those schizophrenic projects situated on the boundary between academic research and practical software development. 

One of the first hurdles I had to overcome was teaching Coccinelle about the macros in my code. Coccinelle has to do all its own parsing and pre-processing -- you can't just run the input C code through cpp because then you'd have to map the post-processor output back to the original code. Macros will sometimes confuse it enough that it gives up parsing a function until it reaches the next safe grammatical starting point (e.g., the next function) -- which may mean that it doesn't process most of the file. To get around this, you can create a list of macros and feed them to spatch with the `-macro_file` option. (Yes, that's one dash -- one of my pet peeves about Coccinelle is the non-standard command-line option style.) For example, here are a few lines from the macro file I used for e2fsprogs: 

```
#define EXT2FS_ATTR(a)
        #define _INLINE_ inline
        #define ATTR(a)
```

You can build the list of macros by hand, but spatch has a feature that helps find them automatically. The `-parse_c` option makes spatch list the top ten parsing errors, which will include the macro name. For example, some of the output from running `spatch -parse_c` on e2fsprogs: 

```
EXT2FS_ATTR: present in 85 parsing errors
        example:
    
              static int check_and_change_inodes(ext2_ino_t dir,
                                          int entry EXT2FS_ATTR((unused)),
                                          struct ext2_dir_entry *dirent, int
                                          offset,
                                          int  blocksize EXT2FS_ATTR((unused)),
```

Coccinelle has improved significantly in the past few weeks. The 0.1.2 release had a number of bugs that made spatch unusable for me. The next release, 0.1.3, fixed those bugs and with it I was able to make practical, real-world patches. The 0.1.4 release will be out shortly. The developers wrote and released more documentation, including a [description of all the command-line options](<http://www.emn.fr/x-info/coccinelle/options.pdf>) [PDF] and a [grammar for SmPL](<http://www.emn.fr/x-info/coccinelle/docs/index.html>). Many more [example spatch scripts](<http://www.emn.fr/x-info/coccinelle/rules/index.html>) are available now. The best reference for learning Coccinelle continues to be the slides from their 2007 OLS tutorial and the associated [paper](<http://www.emn.fr/x-info/coccinelle/ols07-padioleau.pdf>) [PDF]. White space handling is improving; originally Coccinelle didn't care much about white space and frequently mangled transformations involving it, which is a problem if you want to take the hand out of hand-editing. One of my semantic patches left a dangling semi-colon in the middle; the developers sent me a patch to fix it within a few days. 

One thing I am absolutely certain of: learning Coccinelle and writing semantic patches was way more fun than making the changes by hand or using regular expressions. I also had much greater confidence that my changes were correct; it is remarkably pleasant to make several hundred lines of changes and have the result compile cleanly and pass the regression tests the first time. 

### Related work

If you really want to, you can do everything Coccinelle can do by writing your own scripts -- after all, code is code. But you have to deal with all the little corner cases -- e.g., to C, white space is all the same, generally speaking, but regular expressions care intensely about the difference between a space, a newline, and a tab. Use the right tool for the job -- if you're just replacing a variable name and your first script works, great. If you're changing a calling convention or moving the allocation and freeing of an object to another context, give a tool like Coccinelle a try. 

In terms of power and flexibility, Coccinelle is similar to the [Stanford compiler checker](<http://www.stanford.edu/~engler/mc-osdi.pdf>) [PDF] (commercialized by [Coverity](<http://coverity.com>)). While the compiler checker is far more mature and has better flow analysis and parsing, Coccinelle can generate code to fix the bugs it finds. Most importantly, Coccinelle is open source, so developers can find and fix bugs themselves. 

Some IDEs include tools to automatically refactor code, which is one aspect of what Coccinelle does. I have never personally used one of these IDE refactoring tools and can't compare it with Coccinelle, but my friends who have report that their stability leaves something to be desired. [Xrefactory](<http://www.xref.sk/xrefactory/main.html>) is a refactoring tool available on *NIX platforms which is fully integrated with Emacs and XEmacs. It is not open source and requires the purchase of a license, although one version is available for use free of charge. 

### Conclusion

Coccinelle is an open source tool that can analyze and transform C code according to specified rules, or semantic patches. Semantic patches are much more powerful than patches or regular expressions. The tool is beta quality right now but usable for practical tasks and the developers are very responsive. It's worth learning for any developer making a non-trivial interface change.
