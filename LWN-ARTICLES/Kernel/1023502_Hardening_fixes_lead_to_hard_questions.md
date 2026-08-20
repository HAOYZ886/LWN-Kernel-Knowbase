---
title: Hardening fixes lead to hard questions
url: https://lwn.net/Articles/1023502/
date: "June 2, 2025"
category: "Development tools-Git"
author: "By Jonathan Corbet June 2, 2025"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 2, 2025

Kees Cook's ["hardening fixes" pull request](<https://lwn.net/ml/all/202505310759.3A40AD051@keescook>) for the 6.16 merge window looked like a straightforward exercise; it only contained four commits. So just about everybody was surprised when it resulted in Cook being temporarily blocked from his kernel.org account among fears of malicious activity. When the dust settled, though, the red alert was canceled. It turns out, surprisingly, that Git is a tool with which one can inflict substantial self-harm in a moment of inattention. 

Linus Torvalds [reacted strongly](<https://lwn.net/ml/all/CAHk-=wj4a_CvL6-=8gobwScstu-gJpX4XbX__hvcE=e9zaQ_9A@mail.gmail.com>) to Cook's pull request after noticing that many of the commits found within it had been modified in strange ways. Git tracks both the author of a commit (the person who wrote the code), and the committer (the person who put that code into the repository). In this case, there were changes that claimed to have been committed by Torvalds, but they were actually rewritten (but unmodified beyond the metadata) versions of his commits with different SHA IDs. Torvalds said: ""You seem to have actively maliciously modified your tree completely"", implying that some sort of deliberate, underhanded change had been made. He copied kernel.org maintainer Konstantin Ryabitsev, asking that Cook's account there be disabled; Ryabitsev [duly complied](<https://lwn.net/ml/all/20250531-resolute-glittering-cuckoo-b6cd91@lemur>). News quickly spread around the Internet, along with a lot of speculation about possible supply-chain attacks or other malicious activity. 

While use of kernel.org is not mandatory, most kernel maintainers do keep their repositories there. Banishment therefrom will, thus, leave a maintainer unable to do their work; unable, in this case, to even fix the problems that caused that banishment in the first place. It has never been explicitly said that a request from Torvalds is enough to cause a kernel.org account to be disabled, but it is not surprising in retrospect. Still, it must have come as a shock, even without the suggestions of possible malicious activity. 

Cook, though, reacted calmly to his banishment, [saying](<https://lwn.net/ml/all/202505312229.DE917E6D@keescook>) that he had not created the problematic repository intentionally; ""I think I have an established track record of asking you first before I intentionally do stupid things with git"". He [went through the exercise](<https://lwn.net/ml/all/202505312300.95D7D917@keescook>) of recreating that repository, showing all the steps along with data from the Git reflog. In the end, he was able to reproduce the problem with an invocation of the [b4](<https://b4.docs.kernel.org/en/latest/index.html>) tool's [` trailers` subcommand](<https://b4.docs.kernel.org/en/latest/contributor/trailers.html>). 

B4 is a tool that has made life far easier for kernel developers and (especially) maintainers. It handles many of the tasks of applying patches, ensuring that all offered tags ("Reviewed-by" and such) are applied, and more. The `b4 trailers` command, in particular, will look for replies to a set of already-committed patches containing new tags, then rewrite the commit history to include those tags in the changelogs. It is, at its core, a rebasing operation. Those should always be approached with care, but they do not ordinarily lead to this kind of problem. 

In this case, `b4 trailers` advised Cook that it was about to modify 39 commits. By [his own admission](<https://lwn.net/ml/all/202506010833.A33888CC@keescook>), Cook missed that warning and told it to proceed, then used a forced push to upload the resulting repository to kernel.org. Ryabitsev, who is the b4 maintainer, was [willing to share](<https://lwn.net/ml/all/20250601-pony-of-imaginary-chaos-eaa59e@lemur>) the blame: 

> Well, that's the point where the user, in theory, goes "this is weird, why is it 39 commits," and does Ctrl-C, but I'm happy to accept blame here -- we should be more careful with this operation and bail out whenever we recognize that something has gone wrong. 

He added that he was ""100% convinced"" that there was no malicious activity involved. Cook's account was reactivated; he promptly put together a new pull request for the hardening fixes, which was [pulled by Torvalds](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cd2e103d57e5>) on June 1\. 

There will be some changes to b4 to try to prevent this kind of mistake from happening again. Torvalds has [asked](<https://lwn.net/ml/all/CAHk-=wgcQdD0UzMJrNhQuYAC2wgGtfrCry_iokswaEE5j7W9YA@mail.gmail.com>) that it refuse to rewrite commits that were committed by anybody other than the user running the command; Ryabitsev has [agreed](<https://lwn.net/ml/all/20250601-wandering-graceful-crane-ffc0b7@lemur>) to make that change. There will probably be others as well, once the developers involved understand why b4 decided to modify so many commits in this case. 

So this episode appears to have run its course. The real moral of the story, perhaps, is that powerful tools can sometimes have powerfully adverse effects. It can take time — and hard experience with those effects — to learn where the pitfalls are and what types of guard rails need to be installed. We have just seen an example of how that experience is gained.
