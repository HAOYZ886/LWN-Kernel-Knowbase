---
title: The role of LLMs in patch review
url: https://lwn.net/Articles/1064830/
date: "March 31, 2026"
category: "Development tools-Large language models"
author: "By Daroc Alden March 31, 2026"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Daroc Alden**  
March 31, 2026

Discussion of [ a memory-management patch set](<https://lwn.net/ml/all/cover.1773924928.git.ljs@kernel.org/>) intended to clean up a helper function for handling huge pages spiraled into something else entirely after it was posted on March 19\. Memory-management maintainer Andrew Morton [ proposed](<https://lwn.net/ml/all/20260319200917.ce345a369d035050b6329ac5@linux-foundation.org/>) making changes to the subsystem's review process, to require patch authors to respond to feedback from [Sashiko](<https://sashiko.dev/>), the [ recently released LLM-based kernel patch review system](<https://lwn.net/Articles/1063303/#sashiko>). Other sub-maintainers, particularly Lorenzo Stoakes, objected. The resulting discussion about how and when to adopt Sashiko is potentially relevant to many other parts of the kernel. 

Morton began by saying that the current way Sashiko integrates into the memory-management workflow isn't working. He merges patches to his tree, and ""then half a day later a bunch of potential issues are identified."" Morton stated that he was going to further increase the lag between seeing a patch set on the mailing list and merging it to his tree, to give Sashiko time to produce feedback and patch authors time to respond to it. He also wanted its reviews distributed to a wider audience — partly to better determine how useful its comments are, which he is ""paying close attention to"". 

Stoakes [ said](<https://lwn.net/ml/all/8ae5a5a3-f2ad-4be0-89b4-b8beadd957e9@lucifer.local/>) that he would look at the Sashiko reviews for the patch set, but asked Morton to hold off on incorporating it into the subsystem's workflow. He said that he appreciates the tooling, but that it is currently too noisy to use in that way. Stoakes referenced [ his message](<https://lwn.net/ml/all/39e6b4d2-8a30-4eaa-908d-5d11b746f8d5@lucifer.local/>) in the thread introducing Sashiko (that began on March 17) where he expressed the opinion that its false-positive rate was higher than his own experience using [ Chris Mason's kernel-review prompts](<https://lwn.net/Articles/1041694/>). David Hildenbrand [ agreed](<https://lwn.net/ml/all/548f7e8c-6b06-40e9-82e5-e3718d19431f@kernel.org/>) that the false-positive rate was too high to be useful. 

Roman Gushchin, Sashiko's creator, [told](<https://lwn.net/ml/all/87tsu9kgv3.fsf@linux.dev/>) Morton that he was actively working on integrating Sashiko with the kernel's email-based workflow, and that he hoped to have it sending reviews to appropriate recipients within the next week. Morton took the opportunity to [ ask](<https://lwn.net/ml/all/20260320203311.715ed75bcd84c18d24894324@linux-foundation.org/>) about another problem with the tool — that many patch sets sent to the mailing list fail to apply in Sashiko's environment. In a [ follow-up message](<https://lwn.net/ml/all/20260321171530.8b3e8207f89d5a7384b9f01f@linux-foundation.org/>) he expressed his intention not to apply patches to his tree that the system couldn't. Gushchin [explained](<https://lwn.net/ml/all/877br4k3ya.fsf@linux.dev/>) that Sashiko tries to apply patch sets to several bases, in order. For memory-management patches, it uses the patch set's base commit (if specified), then the mm-new tree, followed by mm-unstable, mm-stable, linux-next, and finally Linus Torvalds's tree. The review system evaluates the code in the first tree where the application attempt is successful. He didn't address why mailing-list patches would be failing to apply to these trees, however. 

Stoakes [ asked](<https://lwn.net/ml/all/5cd57c69-7193-422f-b6b5-75bb5234e5f3@lucifer.local/>) Morton to hold off on integrating Sashiko so deeply into his workflow: 

> Andrew, for crying out loud. Please don't do this. 
> 
> 2 of the 3 series I respun on Friday, working a 13 hour day to do so, don't apply to Sashiko, but do apply to the mm tree. 
> 
> I haven't the _faintest clue_ how we are supposed to factor a 3rd party experimental website applying or not applying series into our work?? 

He went on to say that he was not attempting to disrespect Gushchin or his efforts, but that even Gushchin had agreed that the tool was not ready to become a blocking component of the development process. Gushchin [replied](<https://lwn.net/ml/all/87jyv2hw50.fsf@linux.dev/>) to say that working on Sashiko had increasingly shown him the subjectivity of reviews, and the importance of social context in providing good reviews. He acknowledged that it wasn't ""perfect in any way"" but suggested that some level of false positives (for example, 20%) was acceptable from a tool that catches the majority of bugs before they're merged. He suggested that this might be a reasonable lens through which to view Sashiko's current performance and future development. 

Stoakes [ replied](<https://lwn.net/ml/all/bc42fdda-6be2-46ef-bec5-1ae82092f61b@lucifer.local/>) to clarify that he was objecting to Morton's unilateral demand that ""every single point Sashiko raises is responded to"". He was emphatically not blaming Gushchin for failures of the memory-management subsystem's review model, and thought the tool was promising. He reiterated his perception that the tool's false-positive rate was much higher than other people were claiming, and that — given its inexhaustible ability to produce new reviews that require human attention — it was important to think critically about what role it can play in the review process. Incorporating the tool in its present state, as anything other than a simple advisory, would increase the workload on the already overworked memory-management maintainers, he said. 

This sentiment resonated with Pedro Falcato, who [ agreed](<https://lwn.net/ml/all/j6hsn4mxnzoukebcdggu6ojkyaj54joohocbbkrzksm47km7ni@zzp7aurwmto7/>) that Sashiko should remain optional for the time being. Morton [ disagreed](<https://lwn.net/ml/all/20260323143604.603b20aec4e3ab77cabec107@linux-foundation.org/>): 

> Rule #1 is, surely, "don't add bugs". This thing finds bugs. If its hit rate is 50% then that's plenty high enough to justify people spending time to go through and check its output. 
> 
> [...] 
> 
> Look, I know people are busy. If checking these reports slows us down and we end up merging less code and less buggy code then that's a good tradeoff. 

Avoiding bugs is important, Falcato [ agreed](<https://lwn.net/ml/all/4fb7fzdf4uirsxlxiwd4arbhlezgrawb72nm7sl2slntvxlng7@kimmnebrr4c4/>), but: ""I simply don't think either contributors or maintainers will be particularly less stressed with the introduction of obligatory AI reviews."" He suggested that simply codifying the memory-management review process (as the [netdev review process](<https://docs.kernel.org/process/maintainer-netdev.html>) has been) would be more helpful than mandating the use of Sashiko (a suggestion that Mike Rapoport later [supported](<https://lwn.net/ml/all/acJEFArj6uw2Z_2e@kernel.org/>)). Falcato also pointed out that Sashiko is experimental, untested software, and it should probably not be made critical to the process yet on those grounds. 

Morton [ responded](<https://lwn.net/ml/all/20260323170537.0aee4e4906169db510e9893c@linux-foundation.org/>) by looking at actual replies to Sashiko's reviews on the linux-mm mailing list. Out of about 35 emails, 22 received replies indicating that alterations were definitely needed, with the rest being more ambiguous, being false reports, or not being responded to. He expressed the opinion that such a hit rate of finding actual problems in patches was worth the pain of figuring out how to integrate Sashiko into the process. ""That's a really high hit rate! How can we possibly not use this, if we care about Rule #1?"" 

Stoakes [ disagreed](<https://lwn.net/ml/all/95130554-15c4-42dd-bda8-1ee8bc3fa370@lucifer.local/>) with Morton's interpretation of the data, pointing out that those 22 emails indicate cases where the tool was correct in _at least one_ individual observation. Since it normally sends multiple suggestions and questions per review, the actual rate of false positives for individual comments must be substantially worse than that. 

Stoakes again reiterated that he found Sashiko useful, and was using it in his own reviews to some degree. The problem was in making it a mandatory part of the process. He suggested that Morton should delegate the decision of whether and how to use Sashiko to the sub-maintainers, avoid requiring that its automation cleanly applies a patch before accepting it, avoid requiring that every element of its reviews be responded to, and trust sub-maintainers to discard any parts of the reviews that are not both valid and important. 

Sashiko is, at the time of writing, not even a month old. According to its [statistics page](<https://sashiko.dev/#/stats>), it has written over 10,000 reviews in that time, with an average of approximately 3,500 words of output (counting quoted source code) per patch. It is, quite literally, producing words faster than any individual person could reasonably read. But approximately half of those words, according to various different statistics, are about bugs that no human reviewer spotted ahead of time — either due to the difficulty of reviewing complex kernel code, or just due to a lack of infinite time to dedicate to the prospect. And several kernel contributors [ are finding the reviews useful](<https://lwn.net/ml/all/DHCUIGX4URD0.1KCTEYDWLQWE0%40google.com/>). 

As is the case unfortunately often, the problem posed by the use of Sashiko is a social one, not a technical one: how much extra reading, hallucinated problems, delays in the review process, robotic gatekeeping, and reliance on proprietary models is acceptable in order to make sure the kernel accepts less buggy code? It's a question that will eventually touch every subsystem of the kernel and beyond, not just the memory-management code, and one that undoubtedly deserves a lot of discussion. There are no easy answers, but hopefully the kernel community will eventually be able to reach a consensus.
