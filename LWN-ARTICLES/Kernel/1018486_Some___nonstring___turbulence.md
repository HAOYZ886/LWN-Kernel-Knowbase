---
title: Some __nonstring__ turbulence
url: https://lwn.net/Articles/1018486/
date: "April 24, 2025"
category: GCC
author: "By Jonathan Corbet April 24, 2025"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 24, 2025

New compiler releases often bring with them new warnings; those warnings are usually welcome, since they help developers find problems before they turn into nasty bugs. Adapting to new warnings can also create disruption in the development process, though, especially when an important developer upgrades to a new compiler at an unfortunate time. This is just the scenario that played out with the [6.15-rc3 kernel release](<https://lwn.net/ml/all/CAHk-=wgjZ4fzDKogXwhPXVMA7OmZf9k0o1oB2FJmv-C1e=typA@mail.gmail.com/>) and the implementation of `-Wunterminated-string-initialization` in GCC 15\. 

Consider a C declaration like: 

```
char foo[8] = "bar";
```

The array will be initialized with the given string, including the normal trailing NUL byte indicating the end of the string. Now consider this variant: 

```
char foo[8] = "NUL-free";
```

This is a legal declaration, even though the declared array now lacks the room for the NUL byte. That byte will simply be omitted, creating an unterminated string. That is often not what the developer who wrote that code wants, and it can lead to unpleasant bugs that are not discovered until some later time. The `-Wunterminated-string-initialization` option emits a warning for this kind of initialization, with the result that, hopefully, the problem — if there is a problem — is fixed quickly. 

The kernel community has worked to make use of this warning and, hopefully, eliminate a source of bugs. There is only one little problem with the new warning, though: sometimes the no-NUL initialization is exactly what is wanted and intended. See, for example, [this declaration](<https://elixir.bootlin.com/linux/v6.14.3/source/fs/cachefiles/key.c#L11>) from `fs/cachefiles/key.c`: 

```
static const char cachefiles_charmap[64] =
    	"0123456789"			/* 0 - 9 */
    	"abcdefghijklmnopqrstuvwxyz"	/* 10 - 35 */
    	"ABCDEFGHIJKLMNOPQRSTUVWXYZ"	/* 36 - 61 */
    	"_-"				/* 62 - 63 */
    	;
```

This `char` array is used as a lookup table, not as a string, so there is no need for a trailing NUL byte. GCC 15, being unaware of that usage, will emit a false-positive warning for this declaration. There are many places in the kernel with declarations like this; the ACPI code, for example, uses a lot of four-byte string arrays to handle the equally large set of four-letter ACPI acronyms. 

Naturally, there is a way to suppress the warning when it does not apply by adding an attribute to the declaration indicating that the `char` array is not actually holding a string: 

```
__attribute__((__nonstring__))
```

Within the kernel, the macro `__nonstring` is used to shorten that attribute syntax. Work has been ongoing, primarily by Kees Cook, to fix all of the warnings added by GCC 15\. Many patches have been circulated; quite a few of them are in linux-next. Cook has also been working with the GCC developers to improve how this annotation works and to [fix a problem](<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=118095>) that the kernel project ran into. There was some time left to get this job done, though, since GCC 15 has not actually been released — or so Cook thought. 

Fedora 42 _has_ been released, though, and the Fedora developers, for better or worse, decided to include a pre-release version of GCC 15 with it as the default compiler. The Fedora project, it seems, has decided to follow [a venerable Red Hat tradition](</2000/1005/dists.php3>) with this release. Linus Torvalds, for better or worse, decided to update his development systems to Fedora 42 the day before tagging and releasing 6.15-rc3. Once he tried building the kernel with the new compiler, though, things started to go wrong, since the relevant patches were not yet in his repository. Torvalds responded with a series of changes of his own, applied directly to the mainline about two hours before the release, to fix the problems that he had encountered. They included [this patch](<https://git.kernel.org/linus/4b4bd8c50f48>) fixing warnings in the ACPI subsystem, and [this one](<https://git.kernel.org/linus/05e8d261a34e>) fixing several others, including the example shown above. He then tagged and pushed out 6.15-rc3 with those changes. 

Unfortunately, his last-minute changes broke the build on any version of GCC prior to the GCC 15 pre-release — a problem that was likely to create a certain amount of inconvenience for any developers who were not running Fedora 42\. So, shortly after the 6.15-rc3 release, Torvalds tacked on [one more patch](<https://git.kernel.org/linus/9d7a0577c9db>) backing out the breaking change and disabling the new warning altogether. 

This drew [a somewhat grumpy note](<https://lwn.net/ml/all/202504201840.3C1F04B09@keescook>) from Cook, who said that he had already sent patches fixing all of the problems, including the build-breaking one that Torvalds ran into. He asked Torvalds to revert the changes and use the planned fixes, adding: ""It is, once again, really frustrating when you update to unreleased compiler versions"". Torvalds [disagreed](<https://lwn.net/ml/all/CAHk-=whryuuKnd_5w6169EjfRr_f+t5BRmKt+qfjALFzfKQNvQ@mail.gmail.com>), saying that he needed to make the changes because the kernel failed to build otherwise. He also asserted that GCC 15 _was_ released by virtue of its presence in Fedora 42\. Cook [was unimpressed](<https://lwn.net/ml/all/202504210909.D4EAB689@keescook>): 

> Yes, I understand that, but you didn't coordinate with anyone. You didn't search lore for the warning strings, you didn't even check -next where you've now created merge conflicts. You put insufficiently tested patches into the tree at the last minute and cut an rc release that broke for everyone using GCC <15\. You mercilessly flame maintainers for much much less. 

Torvalds [stood his ground](<https://lwn.net/ml/all/CAHk-=whjZ-id_1m7cgp4aC+N6yZj3s5Jy=mf2oiEADJ3Tp8sxw@mail.gmail.com>), though, blaming Cook for not having gotten the fixes into the mainline quickly enough. 

That is where the situation stands, as of this writing. Others will undoubtedly take the time to fix the problems properly, adding the changes that were intended all along. But this course of events has created some bad feelings all around, feelings that could maybe have been avoided with a better understanding of just when a future version of GCC is expected to be able to build the kernel. 

As a sort of coda, it is worth saying that Torvalds also has a fundamental disagreement with how this attribute is implemented. The `__nonstring__` attribute applies to variables, not types, so it must be used in every place where a `char` array is used without trailing NUL bytes. He would rather annotate the type, indicating that every instance of that type holds bytes rather than a character string, and avoid the need to mark rather larger numbers of variable declarations. But that is not how the attribute works, so the kernel will have to include `__nonstring` markers for every `char` array that is used in that way.
