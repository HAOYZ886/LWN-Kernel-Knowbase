---
title: "Where have all the reviewers gone?"
url: https://lwn.net/Articles/199110/
date: "September 11, 2006"
category: "Development model-Code review"
author: 
---

One of the often-proclaimed advantages of the free software development model is that of peer review. Our code, we claim, is better because it has been reviewed and improved by a variety of people beyond the original author(s). Reviewers, with their unique perspective, will find bugs and generally help new code fit properly into an existing project. This review process is seen as being so important that a number of projects will not accept code until it has been picked over by other developers. 

So reviewers are a fundamental part of the process. They are also, it seems, somewhat scarce. Consider a couple of examples: 

  * In the kernel space, the reiser4 filesystem has been held up for some time. There are many reasons for that delay, but one of those has been the lack of a thorough review by somebody who understands the Linux virtual filesystem layer well. Greg Kroah-Hartman, in [his OLS keynote](<http://www.kroah.com/log/linux/ols_2006_keynote.html>), said, more generally: ""The big problem ... is we really only have a very small group of people reviewing code in the kernel community."" 

  * The PostgreSQL developers have been engaged in a lengthy discussion on the upcoming 8.2 release, why it is taking as long as it is, and why this release appears (to them) to have little in the way of exciting new features. The conversation has touched on various aspects of that project's development process; there are many things for those developers to think about. One of them, though, as [expressed](<https://lwn.net/Articles/199113/>) by one of the participants, is: ""...the real problem seems to be we do not have enough patch reviewers."" 

If we truly believe that code review is a crucial part of the free software process (and, for the most part, it is likely that we do believe this), then the idea that projects are being slowed by the lack of reviewers is a bit worrying. At best, a reviewer shortage will be a bottleneck in the process; a worse possibility is that some projects will simply decide to do without. 

Reviewers serve a number of purposes. They can often immediately spot that bug that the developer has stared at for hours without finding. If the code is hard to understand, the reviewers will be the first to notice. If the associated documentation is incorrect or (as is more often the case) absent, the reviewers will notice that as well. When code appears to have been written using some sort of specialized, non-public knowledge, reviewers can inquire as to its provenance. Coding style issues, API misuse, inefficient algorithms, use of outdated interfaces, and more can be caught in the review process before the code hits the project's mainline. Reviewers really do increase a project's code quality and long-term maintainability. 

The problem is that code review can be a difficult, tiring, and thankless job. Human nature being what it is, people will often show less than the appropriate amount of gratitude when a reviewer points out their mistakes in public. This is especially true if the code has problems which will require significant amounts of work to fix. The reviewer did not create these problems, he or she is simply the messenger with the bad news. So reviewers tend to get grumpy, especially when they see the same mistakes being made over and over again. 

Developers get credit for their work, in various forms. It is a rare project release, however, which publicly acknowledges those who reviewed the code. Given that writing code is not only a more visible activity, but it also tends to be more fun than reviewing code written by others, it is not surprising that many developers choose to concentrate on their own work. 

Finally, reviewing code can be intimidating - especially if the patch of interest has a Big Name behind it. Many potential reviewers may feel that they simply do not have the standing to poke at other peoples' work. The fact is, however, that even people with a relatively small amount of experience can provide useful reviews, and learn from the process. From Greg's OLS keynote: 

When you are learning to play an instrument, you don't start out writing full symphonies on your own, you spend years reading other peoples scores, and learning how things are put together and work and interact. Only later do you start writing your own music, small tunes, and then, if you want, working up to bigger pieces. The same goes for programming. You can learn a lot from reading and understanding other people's code. Study the things posted, and ask why things are done specific ways, and point out problems that you have noticed. 

If we want to create the best free systems we can, we must ensure that the review portion of the process does not get slighted. To that end, people who have the requisite skills would do well to dedicate a bit of their time to reviewing code in a project that interests them. Buy a reviewer a beer, and forgive them if they tell you, in front of hundreds or thousands of developers, that your work is best suited for a place in the project's "bad examples" repository. Listen to what the reviewers say, respond to it, and thank them. The result will be better software for all of us.
