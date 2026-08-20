---
title: Supporting kernel development with large language models
url: https://lwn.net/Articles/1026558/
date: "June 26, 2025"
category: "Development tools-Large language models"
author: "By Jonathan Corbet June 26, 2025 OSSNA"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
June 26, 2025

* * *

[OSSNA](<https://lwn.net/Archives/ConferenceIndex/#Open_Source_Summit_North_America-2025>)

Kernel development and machine learning seem like vastly different areas of endeavor; there are not, yet, stories circulating about the vibe-coding of new memory-management algorithms. There may well be places where machine learning (and large language models — LLMs — in particular) prove to be helpful on the edges of the kernel project, though. At the [2025 North-American edition of the Open Source Summit](<https://events.linuxfoundation.org/open-source-summit-north-america/>), Sasha Levin presented some of the work he has done putting LLMs to work to make the kernel better 

[![\[Sasha Levin\]](https://static.lwn.net/images/conf/2025/ossna/SashaLevin-sm.png)](<https://lwn.net/Articles/1026565/>) An LLM, he began, is really just a pattern-matching engine with a large number of parameters; it is a massive state machine. Unlike the sort of state machine typically seen in the kernel, though, LLMs perform state transitions in a probabilistic, rather than deterministic, manner. Given a series of words, the LLM will produce a possible next word in the sequence. Given ""the Linux kernel is written in..."", the LLM will almost certainly respond "C". There is a much lower probability, though, that it might say "Rust" or "Python" instead. 

An LLM works with a "context window", which is the user-supplied text it can remember while answering questions. A system like [Claude](<https://en.wikipedia.org/wiki/Claude_%28language_model%29>) has a context window of about 200,000 tokens, which is enough for an entire kernel subsystem. 

Levin does not believe that LLMs will replace humans in tasks like kernel development. Instead, an LLM should be viewed as the next generation of fancy compiler. Once upon a time, developers worked in assembly; then higher-level languages came along. Some sneered at this new technology, saying that "real developers" did their own register allocation. But, in time, developers adopted better programming languages and became more productive. An LLM is just another step in that direction; it is not a perfect tool, but it is good enough to improve productivity. 

#### LLM-generated code in the kernel

As an example, he pointed to [a patch credited to him](<https://git.kernel.org/linus/391dda1bd7c5>) that was merged for the 6.15 release. That patch was entirely written by an LLM, changelog included. Levin reviewed and tested it, but did not write the code. This fix, he said, is a good example of what LLMs can do well; they excel at small, well-defined tasks, but cannot be asked to write a new device driver. LLMs also help with writing the commit message, which is often more difficult than writing the patch itself, especially for developers whose native language is not English. 

He pointed out a couple of things about the patch itself, excerpted here: 

```
-/* must be a power of 2 */
        -#define EVENT_HASHSIZE	128
        +/* 2^7 = 128 */
        +#define EVENT_HASH_BITS 7
```

The switch from one hash API to another required specifying a size as a power of two rather than a straight number of bits; the LLM took that into account and made the appropriate change. It also realized, later in the patch, that a masking operation was not needed, so it took that operation out. The LLM, he said, generated code that was both correct and efficient. 

Another example is the [`git-resolve` script](<https://elixir.bootlin.com/linux/v6.16-rc3/source/scripts/git-resolve.sh>) that was merged for 6.16. This script, which came out of a [late 2024 discussion](<https://lwn.net/Articles/1001526/>) on ambiguous commit IDs, will resolve an ambiguous (or even incorrect) ID into a full commit. It, too, was generated with an LLM. Not only does it work, but it includes a full set of self tests, something he noted (with understatement) is unusual for code found in the kernel's `scripts` directory. LLMs, he said, ""won't give you a frowny face"" when asked to generate tests. The script includes documentation (also unusual for that directory), and is being used on a daily basis in the kernel community. 

Moving on, he introduced the concept of "embeddings", which are a way of representing text within an LLM. They can be thought of as an equivalent to a compiler's internal representation of a program. Embeddings turn human language into vectors that can be processed mathematically. They preserve the semantic meaning of the text, meaning that phrases with similar meanings will "compile" to similar embeddings. That, in turn, allows meaning-based searching. In the kernel context, embeddings can help in searching for either commits or bugs that are similar to a given example. 

Another useful LLM technology is "retrieval augmented generation" (RAG). LLMs, he said, have an unfortunate tendency to make things up when they do not know the answer to a question; an LLM will only rarely admit that it does not know something. That can be ""really annoying"" for generated code; an LLM will make up kernel functions that do not exist, for example. RAG works to ground an LLM in actual knowledge, enabling the model to look up information as needed, much like how humans use documentation. It is also useful to update an LLM with knowledge that came about after its training was done. 

For the kernel in particular, RAG can ground the model and teach it about kernel-specific patterns. It also adds explainability, where the model can cite specific examples to justify the decisions it makes. Among other things, RAG allows the model to connect to a Git repository, giving it access to the kernel's development history. 

#### Updates and CVEs

The stable kernels include massive number of patches that have been backported from the mainline; the 5.10 series, for example, has incorporated over 31,000 commits after the initial 5.10 release was made. Maintaining these stable updates requires reviewing around 100 patches per day — every day, with no breaks. Of those, maybe five or ten are suitable for backporting. It is a tedious and frustrating process that does not scale; as a result, important fixes are sure to fall through the cracks. 

The "AUTOSEL" tool has been around for some years; it tries to select the mainline commits that should be considered for backporting. The initial version was primitive; it would just look for specific keywords in the changelog. Switching AUTOSEL to an LLM causes it to act like ""another stable-kernel maintainer"", albeit a special one who remembers every backporting decision that has ever been made. It works by creating an embedding for every commit in the history, then finding similarities with new commits that may be solving the same kind of problem. 

AUTOSEL, he noted, is not replacing the stable maintainers, but it does narrow down the set of commits that they must consider. It is able to process hundreds of commits quickly, catching fixes that humans will miss. It also explains its reasoning in each email that is sent to the list ([random example](<https://lwn.net/ml/all/20250617122338.1969838-6-sashal@kernel.org/>)) proposing a patch for backporting. When asked to consider a specific commit, he said, AUTOSEL can also recommend similar commits for consideration. 

People ask which LLM is being used for AUTOSEL; the answer is ""all of them"". Each model has its own strengths and weaknesses, so AUTOSEL asks several of them, then allows each to vote on the conclusion. If enough models vote in favor of a backport, it is referred to the humans for consideration. 

In early 2024, the kernel project [took on the responsibility](<https://lwn.net/Articles/961978/>) for assigning its own CVE numbers. The tooling to support this work started as a collection of ""bash hacks"" that quickly became unmaintainable. So the CVE team decided to convert them to Rust, since ""that's what the cool kids do these days"". The only problem is that the CVE team members are all kernel developers who are not that proficient in Rust. LLMs _are_ proficient in the language, though, and were able to quickly rewrite the scripts, adding documentation and tests in the process. The new scripts are more maintainable and vastly more efficient. 

The CVE process itself is a challenge similar to that of backporting; commits must be reviewed for security relevance, which is another tedious task. It is hard to find people with the requisite expertise to do this work; the people with the needed skills can easily find more rewarding work to do. A purely human-based process thus runs behind, misses important vulnerabilities, while occasionally flagging bugs that are not, in fact, vulnerabilities. 

This is, in other words, another job for a machine. The CVE selection is able to share much of the infrastructure used by AUTOSEL, but this time the LLM is being asked to look for commits that somehow resemble previous vulnerability fixes. 

He concluded by saying that, using LLMs, the kernel community now has a system that can make use of multiple models, directly access Git repositories, and make use of historical data to answer various types of questions about kernel patches. He provided URLs for [AUTOSEL](<https://git.sr.ht/~sashal/autosel>) and [the commit classifier](<https://git.sr.ht/~sashal/commit-classifier>). 

Tim Bird asked whether there is a risk of humans trusting the output from the LLMs too much, allowing errors to creep in. Levin agreed that LLMs can be wrong, but he said that humans can be wrong too, and they often are. Another participant asked about the licensing for code that is emitted by an LLM; Levin said that he has not really thought about the problem, and assumes that, if an LLM produces code, he is free to make use of it. 

The last question was whether this infrastructure could be used to examine patches prior to merging in the hope of catching bugs earlier. This is an area that Levin has [explored in the past](<https://lwn.net/Articles/803695/>), but that is not a focus currently. He agreed that LLMs could do that work, but it would be a huge job, and LLMs are still too expensive to use in that way. Perhaps in the future, he said, when the price has fallen, that sort of analysis will be possible. 

[Thanks to the Linux Foundation for supporting our travel to this event.]
