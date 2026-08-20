---
title: Andrew Morton on kernel development
url: https://lwn.net/Articles/285088/
date: "June 11, 2008"
category: Development model
author: "By Jonathan Corbet June 11, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 11, 2008

Andrew Morton is well-known in the kernel community for doing a wide variety of different tasks: maintaining the -mm tree for patches that may be on their way to the mainline, reviewing lots of patches, giving presentations about working with the community, and, in general, handling lots of important and visible kernel development chores. Things are changing in the way he does things, though, so we asked him a few questions by email. He responded at length about the -mm tree and how that is changing with the advent of linux-next, kernel quality, and what folks can do to help make the kernel better. 

Years ago, there was a great deal of worry about the possibility of burning out Linus. Life seems to have gotten easier for him since then; now instead, I've heard concerns about burning out Andrew. It seems that you do a lot; how do you keep the pace and how long can we expect you to stay at it? 

I do less than I used to. Mainly because I _have_ to - you can't do the same thing at a high level of intensity for over five years and stay sane. 

I'm still keeping up with the reviewing and merging but the -mm release periods are now far too long. 

There are of course many things which I _should_ do but which I do not. 

Over the years my role has fortunately decreased - more maintainers are running their own trees and the introduction of the linux-next tree (operated by Stephen Rothwell) has helped a lot. 

The linux-next tree means that 85% of the code which I used to redistribute for external testing is now being redistributed by Stephen. Some time in the next month or two I will dive into my scripts and will find a way to get the sufficiently-stable parts of the -mm tree into linux-next and then I will hopefully be able to stop doing -mm releases altogether. 

So. The work level is ramping down, and others are taking things on. 

What can we do to help? 

I think code review would be the main thing. It's a pretty specialised function to review new code well. The people who specialise in the area which the new code is changing are the best reviewers but unfortunately I will regularly find myself having to review someone else's stuff. 

Secondly: it would help if people's patches were less buggy. I still have to fix a stupidly large number of compile warnings and compilation errors and each -mm release requires me to perform probably three or four separate bisection searches to weed out bad patches. 

Thirdly: testing, testing, testing. 

Fourthly: it's stupid how often I end up being the primary responder on bug reports. I'll typically read the linux-kernel list in 1000-email batches once every few days and each time I will come across multiple bug reports which are one to three days old and which nobody has done anything about! And sometimes I _know_ that the person who is responsible for that part of the kernel has read the report. grr. 

Is it your opinion that the quality of the kernel is in decline? Most developers seem to be pretty sanguine about the overall quality problem. Assuming there's a difference of opinion here, where do you think it comes from? How can we resolve it? 

I used to think it was in decline, and I think that I might think that it still is. I see so many regressions which we never fix. Obviously we fix bugs as well as add them, but it is very hard to determine what the overall result of this is. 

When I'm out and about I will very often hear from people whose machines we broke in ways which I'd never heard about before. I ask them to send a bug report (expecting that nothing will end up being done about it) but they rarely do. 

So I don't know where we are and I don't know what to do. All I can do is to encourage testers to report bugs and to be persistent with them, and I continue to stick my thumb in developers' ribs to get something done about them. 

I do think that it would be nice to have a bugfix-only kernel release. One which is loudly publicised and during which we encourage everyone to send us their bug reports and we'll spend a couple of months doing nothing else but try to fix them. I haven't pushed this much at all, but it would be interesting to try it once. If it is beneficial, we can do it again some other time. 

There have been a number of kernel security problems disclosed recently. Is any particular effort being put into the prevention and repair of security holes? What do you think we _should_ be doing in this area? 

People continue to develop new static code checkers and new runtime infrastructure which can find security holes. 

But a security hole is just a bug - it is just a particular type of bug, so one way in which we can reduce the incidence rate is to write less bugs. See above. More careful coding, more careful review, etc. 

Now, is there any special pattern to a security-affecting bug? One which would allow us to focus more resources on preventing that type of bug than we do upon preventing "average" bugs? Well, perhaps. If someone were to sit down and go through the past five years' worth of kernel security bugs and pull together an overall picture of what our commonly-made security-affecting bugs are, then that information could perhaps be used to guide code-reviewers' efforts and code-checking tools. 

That being said, I have the impression that most of our "security holes" are bugs in ancient crufty old code, mainly drivers, which nobody runs and which nobody even loads. So most metrics and measurements on kernel security holes are, I believe, misleading and unuseful. 

Those security-affecting bugs in the core kernel which affect all kernel users are rare, simply because so much attention and work gets devoted to the core kernel. This is why the recent splice bug was such a surprise and head-slapper. 

I have sensed that there is a bit of confusion about the difference between -mm and linux-next. How would you describe the purpose of these two trees? Which one should interested people be testing? 

Well, things are in flux at present. 

The -mm tree used to consist of the following: 

  * 80-odd subsystem maintainer trees (git and quilt), eg: scsi, usb, net.
  * various patches which I picked up which _should_ be in a subsystem maintainer's tree, but which for one of various reasons didn't get merged there. I spend a lot of time acting as backup for leaky maintainers.
  * patches which are mastered in the -mm tree. These are now organised as subsystems too, and I count about 100 such subsystems which are mastered in -mm. eg: fbdev, signals, uml, procfs. And memory management.
  * more speculative things which aren't intended for mainline in the short-term, such as new filesystems (eg reiser4).
  * debugging patches which I never intend to go upstream.

The 80-odd subsystem trees in fact account for 85% of the changes which go into Linux. Pretty much all of the remaining 15% are the only-in-mm patches. 

Right now (at 2.6.26-rc4 in "kernel time"), the 80-odd subsystem trees are in linux-next. I now merge linux-next into -mm rather than the 80-odd separate trees. 

As mentioned previously, I plan to move more of -mm into linux-next - the 100-odd little subsystem trees. 

Once that has happened, there isn't really much left in -mm. Just 

  * the patches which subsystem maintainers leaked. I send these to the subsystem maintainers.
  * the speculative not-for-next-release features
  * the not-to-be-merged debugging patches.

Do you have any specific goals for the development of the kernel over the next year or so? What would they be? 

Steady as she goes, basically. 

I keep on hoping that kernel development in general will start to ramp down. There cannot be an infinite number of new features out there! Eventually we _should_ get into more of a maintenance mode where we just fix bugs, tweak performance and add new drivers. Famous last words. 

And it's just vaguely possible that we're starting to see that happening now. I do get a sense that there are less "big" changes coming in. When I sent my usual 1000-patch stream at Linus for 2.6.26 I actually received an email from him asking (paraphrased) "hey, where's all the scary stuff?" 

In the early-May discussions, Linus said a couple of times that he does not think code review helps much. Do you agree with that point of view? 

Nope. 

How would you describe the real role of code review in the kernel development process? 

Well, it finds bugs. It improves the quality of the code. Sometimes it prevents really really bad things from getting into the product. Such as rootholes in the core kernel. I've spotted a decent number of these at review time. 

It also increases the number of people who have an understanding of the new code - both the reviewer(s) and those who closely followed the review are now better able to support that code. 

Also, I expect that the prospect of receiving a close review will keep the originators on their toes - make them take more care over their work. 

There clearly must be quite a bit of communication between you and Linus, but much of it, it seems, is out of the public view. Could you describe how the two of you work together? How are decisions (such as when to release) made? 

Actually we hardly ever say anything much. We'll meet face-to-face once or twice a year and "hi how's it going". 

We each know how the other works and I hope we find each other predictable and that we have no particular issues with the other's actions. There just doesn't seem to be much to say, really. 

Is there anything else you would like to say to LWN's readers? 

Sure. Please do contribute to Linux, and a great way of doing that is to test latest mainline or linux-next or -mm and to report on any problems which you encounter. 

Nothing special is needed - just install it on as many machines as you dare and use them in your normal day-to-day activities. 

If you do hit a bug (and you will) then please be persistent in getting us to fix it. Don't let us release a kernel with your bug in it! Shout at us if that's what it takes. Just don't let us break your machines. 

Our testers are our greatest resource - the whole kernel project would grind to a complete halt without them. I profusely thank them at every opportunity I get :) 

We would like to thank Andrew for taking time to answer our questions.
