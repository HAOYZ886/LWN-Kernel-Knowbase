---
title: "The LPC Android microconference, part 2"
url: https://lwn.net/Articles/657139/
date: "September 14, 2015"
category: Android
author: "September 14, 2015 This article was contributed by Nathalie Chan King Choy and John Stultz Linux Plumbers Conference"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

September 14, 2015

This article was contributed by Nathalie Chan King Choy and John Stultz

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2015-Linux_Plumbers_Conference>)

The Linux Plumbers Android microconference was held in Seattle on August 20th. It included discussions of a variety of topics, many of which need to be coordinated within the Android ecosystem. The microconference was split up into two separate sessions; this summary covers the second session, which was held for three hours in the evening. Topics were toybox in Android, improving AOSP vendor trees, providing per-task quality of service, and improving big.LITTLE on Android.

The first session's summary can be found [here](<https://lwn.net/Articles/656324/>).

#### Toybox in Android

Rob Landley started the second half of the microconference with some background on the [toybox project](<http://landley.net/toybox/>), which has recently been [included](<https://lwn.net/Articles/629362/>) in the Android Open Source Project (AOSP), replacing some components of [Android's toolbox](<http://elinux.org/Android_toolbox>). Landley described his early attempts to learn how to build a distribution from scratch, starting in 1999, which he would use to create his [Aboriginal Linux project](<http://landley.net/aboriginal/about.html>). He picked up the [BusyBox tool](<http://www.busybox.net/>) for his project and, after improving it, became the maintainer.

It was after the GPLv3 came out that the trouble started, Landley said. Some code contributed to BusyBox was GPLv2-only, which caused trouble with relicensing. So efforts to audit the code, then remove and replace any GPLv2-only submissions, were made. He became disillusioned with what he saw as the "breaking" of "the GPL", as no longer was there just "the GPL", and it was no longer a universal receiver of code. In the past, most open-source licenses were GPL-compatible and could be up-converted to GPL as needed. With the introduction of GPLv3, along with lots of GPLv2-only code in existence, suddenly this was no longer the case. He handed over BusyBox maintainership to the best person he knew for the job, and started playing with toybox, which he wrote under what he calls a ["BSD zero-clause" license](<http://www.landley.net/toybox/license.html>)—basically public-domain—which would allow the code to be compatible with any other license.

However, soon after starting toybox, he mothballed the project. This was mostly due to the fact that he liked the new BusyBox maintainer and didn't want to undermine the project by leaving and immediately starting a competitor. Also BusyBox had a ten-year head start, which made it intimidating to try to catch up.

Landley then started talking about a [paper he had co-written a decade ago](<http://catb.org/esr/writings/world-domination/world-domination-201.html>) that described how big transitions in computing are always opportunities, since big established players are often unseated by smaller upstarts. The paper was focused on the transition to 64-bit systems and how this was an opportunity for Linux to unseat Windows. 

Landley realized that the transition to mobile was a similar major transition, and Apple's iOS was likely to eventually unseat existing workstations if it were to become dominant. He saw Android as thus becoming an important vehicle for the preservation of a non-proprietary future. His history with Aboriginal Linux made him interested in trying to make Android self-hosting. Unfortunately, due to [Android not accepting any GPL code in user space](<https://source.android.com/source/licenses.html>), BusyBox would not be able to be used, but toybox could be. Thus, he restarted development on toybox. 

At this point, Karim Yaghmour had a few questions, the first being: how far is toybox currently away from BusyBox? Landley pointed to a [online status page](<http://landley.net/toybox/status.html>) that explains (with a somewhat complicated key) what parts are left to do. There is also a [roadmap](<http://landley.net/toybox/roadmap.html>), which helps show the scope of what the project is trying to achieve. That list includes: POSIX-2008, Linux Standard Base (LSB), a self-hosting environment, as well as fully replacing Android's toolbox and Tizen's core tools. 

Yaghmour also asked which commands from toybox have not yet replaced the equivalents in Android's toolbox. Again, he referred people to the status page, but Landley also pointed out that you can look in AOSP at the [toolbox `Android.mk` file](<https://android.googlesource.com/platform/system/core/+/master/toolbox/Android.mk>), which will clarify which non-toybox tools are still being used. Landley noted that Elliott Hughes of the Android team has been helping with some of the testing of toybox. Hughes syncs upstream toybox to the AOSP tree every two weeks. 

At this point the talk ran out of time.

#### Improving vendor AOSP repositories

The next talk ([slides [PDF]](<http://linuxplumbersconf.org/2015/ocw//system/presentations/2895/original/Improving%20AOSP%20Vendor%20Trees.pdf>)) was a discussion led by John Stultz from Linaro on some of the issues he's seen in various vendor trees, and what might be done to improve things. Most vendors utilize a fork-and-try-to-forget model with AOSP, targeting each device with its own tree. Linaro isn't any different, really, as it maintains quite a number of separate AOSP trees for efforts like [96board](<https://www.96boards.org/>) support, work on [Project Ara](<http://www.projectara.com/>), as well as for other projects. This behavior is problematic, since it makes operating system updates more complicated, so vendor device updates suffer. That results in real security difficulties, as recently seen with the [Stagefright issue](<https://lwn.net/Articles/652728/>).

Additionally, devices do tend to include functionality from a number of different vendors: a system-on-chip (SoC) from one vendor, Bluetooth and WiFi from another, sensors from another, etc. As noted in previous talks in the microconference, integrating vendor hardware abstraction layers (HALs) into an AOSP repository is usually a non-trivial process. HALs do not integrate into the tree in a uniform way and some vendor HALs require tweaks to the framework layer. This all complicates things further when it comes time to handle a release update, since there are now multiple parties that are being depended on for updates, which all have to be integrated together.

Another area of pain is the build system. The `device.mk` and `BoardConfig.mk` files used to describe the device usually contain a large set of global build variables, which have no expressions of inter-dependencies (for example, enabling `BOARD_HAVE_BLUETOOTH_BCM` won't do anything if one forgets to enable `BOARD_HAVE_BLUETOOTH`), and the `device.mk` files that list out the `PRODUCT_PACKAGES` to be included have a ton of duplication from device to device. In fact, it seems most vendors create mid-layer common directories and have logic to try to share some of the standard entries for different devices, just to avoid the heavy duplication required. 

AOSP is not really structured to be able to host community contributions, which is another issue. Google has reasonably limited AOSP to hosting device-specific code only for Nexus devices, which the company is able to test and maintain. While this is a understandable and practical decision for Google, it keeps vendors and others in the community working in their own private trees. Since there's no reason to submit code, this results in a limited culture of review, so there isn't necessarily a community sense for what is good or bad code. 

Further, there's a missing sense of best-practices. This may not be true for some vendors who are closer partners with Google, but for the wider community that can't attend the private boot-camp meetings, it is. There is a lack of documentation, so things like HAL integration approaches often end up being done in a cargo-cult manner. Improving documentation and having better examples are areas that need work.

After running through the issues, the discussion was opened up to try to see what could be done to improve things, and not just from the "Google should do X" angle, but also what the community-at-large could potentially implement.

One question was "should HALs be submitted to AOSP?", but since AOSP doesn't accept non-Nexus device support, this didn't seem appropriate. Additionally, many HALs are completely proprietary, so licensing issues would prevent that from happening. There is also the fact that vendors are quite often focused on shipping products, so it's not clear that, even if the code was welcome to be submitted for review, many vendors would take the extra time required to deal with the feedback of that code review. So the angle of finding processes to make things easier for the vendors should be considered.

There was also a suggestion that the community create some space where code that couldn't go into AOSP be collected. This might be a possibility, but it would be good to avoid the "[creating another fork to solve all the forks](<https://xkcd.com/927/>)" sort of solution.

One idea for the build system is to try to reduce the amount of duplicative code in the device directories with something like Kconfig. This would help express configuration dependencies, reducing the number of options required to be specified, and ideally make it easier to build for multiple devices with only a change to the configuration file. Samuel Ortiz mentioned that Intel basically does this, though it doesn't have the dependency tracking. It uses a configuration file to define the device and some common infrastructure in the tree processes that file. Stultz noted that many vendors have something like this to make it easier to build multiple devices; it points to something that may need to be shared generically.

Dmitry Shmidt of Google's Android team asked how Linaro handles doing validation for devices it doesn't have access to. The answer was that it doesn't, and it's a problem. However, another point was raised that the Linux kernel deals with this all the time, since people don't have test machines for all architectures, and it's handled by delegating testing responsibility to architecture maintainers. It was noted that a perspective from the Chrome OS folks might be useful, as they've been able to delegate testing for a wider array of devices.

Rom Lemarchand from Google mentioned that he would like to see more vendors submit code to AOSP. But some attendees said that it is hard to get patches reviewed on Gerrit. A number of folks in the room agreed, saying they had run into similar problems. The Google developers said that it sounded like maybe the auto-adding of maintainers in Gerrit was broken. They promised to look into it and get it solved quickly.

While Google controls commit access to AOSP, its [Gerrit](<https://android-review.googlesource.com/>) installation allows anyone to review and comment on proposed changes. It was proposed that a group of non-Google folks could make a pointed effort to review patches submitted to Gerrit, which would help build up a better community sense of code taste and might even lead to growing external maintainers. Lemarchand mentioned that it would also be nice because it would allow the Google team to better understand whose reviews could be trusted in the community.

The discussion sort of dwindled at this point, so Stultz suggested moving on to the next talk.

#### Providing per-task quality of service

Next Juri Lelli from ARM talked ([slides [PDF]](<https://linuxplumbersconf.org/2015/ocw//system/presentations/3063/original/lelli_slides.pdf>)) about his work on the [energy-aware scheduler](<https://lwn.net/Articles/602479/>) (EAS) and the potential for use of [deadline scheduling](<https://lwn.net/Articles/575497/>) in Android. He started with an overview of the EAS, describing how its focus is on per-entity load tracking and per-entity utilization tracking, which try to size up tasks so that they can be properly placed on the right core in an asymmetric multi-processing environment. The goal is to use as little energy as possible while still getting good performance. When the system is not over-utilized, the scheduler will try to pack tasks onto a single CPU, but once a tipping point is crossed and that one CPU is overloaded, it falls back to the conventional approach of spreading the load around.

He described how EAS ties more of the CPU-power logic like [dynamic voltage and frequency scaling](<https://lwn.net/Articles/466230/>) (DVFS) together into the scheduler, by allowing the scheduler to trigger cpufreq governors directly and make decisions about the speed of each CPU that is being scheduled. This allows for the scheduler to ramp up a processor's frequency to increase capacity if it wants to add a task to that CPU. He also talked a little bit about [schedtune](<https://lwn.net/Articles/655479/>), which provides a single sysfs knob to boost performance on a global or per-cgroup basis. 

At this point it was noted that most of the EAS discussion focused on energy, while the interactive cpufreq governor in Android is mostly focused on latency, so it was asked: how does latency come into the picture and how can it be controlled? Lelli noted that the interactive cpufreq governor tends to boost frequency quickly to provide interactive latency benefits to the user. The schedtune sysfs knob allows for similar boosting, so when an event occurs Android user space could provide some short-term boosting to improve latency response. Riley Andrews from Google noted that the interactive cpufreq governor gives this benefit without user space having to do anything. Though Lelli pointed out that the governor does it a bit blindly, so this allows a more informed and focused response that could save power.

Andrews was also curious about all the various knobs that were available via EAS. Lelli noted that the out-of-tree [HMP scheduler](<https://lwn.net/Articles/541005/>), used by some vendors for big.LITTLE machines, had way too many knobs and was very difficult to tune. So, with EAS, more of the logic has been kept internal in the scheduler so it is easier to get it right. Andrews also wondered about the heuristics for task placement. Lelli noted that there are heuristics and the scheduler needs to make a guess and try to compute the difference in energy used so it can try to give the best tradeoff in performance and energy usage. It was suggested that for more information, folks attend the EAS microconference session that would be going on the next day.

Lelli then switched to the deadline scheduler, which he has been maintaining with others. He mentioned some of the benefits of `SCHED_DEADLINE` over other policies like `SCHED_FIFO`, like how it avoids starvation issues that `SCHED_FIFO` can have. It works using a resource-reservation system that guarantees that a task will get a specific amount of CPU time in a given period. The scheduler just needs to be provided the runtime, period, and deadline values; then it will indicate if it can achieve those constraints or not.

He showed graphs of the performance implications of this model for tasks like movie playback and wanted to know if folks were interested in using this for Android. His thought was that SurfaceFlinger or AudioFlinger might be able to use this policy. One question that came up was how does the deadline scheduler's reservation system handle CPU frequency changes. Lelli replied that it doesn't at the moment and that there needs to be some integration of the policies, so that the deadline scheduler can specify a minimum frequency in order to guarantee that the deadlines are met.

There was also some discussion on how it would interact with `SCHED_FIFO` tasks. It was clarified that `SCHED_DEADLINE` runs at a higher priority than `SCHED_FIFO`, but it can be used so that you can, for example, give a guaranteed limit of 10% of runtime to SurfaceFlinger. That means you don't have to deal with starvation issues, which had been [mentioned as being a problem](<https://source.android.com/devices/audio/latency_contrib.html>) in making the audio pipeline run as a `SCHED_FIFO` task at Google I/O. Andrews said that he had looked at it for SurfaceFlinger earlier, but had some issues with the interface and the cpufreq issue was problematic as well. There were a few more questions, but it was suggested that they be brought up at the EAS microconference the next day.

#### Improving big.LITTLE on Android

Todd Kjos (who was standing in for Tim Murray) from Google's Android team then started reviewing the team's experience dealing with big.LITTLE devices ([slides [PDF]](<http://www.linuxplumbersconf.org/2015/ocw//system/presentations/3399/original/LPC-tkjos.pdf>)). For the most part, big.LITTLE issues and the complex tuning required have been left to the vendors to sort out, as Google hasn't directly addressed it so far. However, that's starting to change. He showed a few examples of the complexities of different styles of asymmetric CPUs that they have seen, from more standard big.LITTLE pairings to more complicated ones, where there might be different sets of "little" CPUs running at different speeds, in addition to the "big" CPUs.

He showed some graphs of the power-performance curves for the different CPUs, which may have cross-over points where it becomes obvious it's worth jumping from the smaller CPUs to the bigger ones. However, he also showed charts where there might be a gap between the two curves, making tuning much more complicated, since to gain any performance at that gap, you have to make a much bigger jump in power consumption.

He described some of the changes in Android M to help. Android keeps track of which tasks are foreground and which tasks are background tasks, so it now uses [cpusets](<http://lwn.net/Articles/55001/>) to help pin all background tasks to the small CPU or CPUs.

Bobby Batacharia from ARM asked if all asynchronous tasks are considered background tasks. Kjos and Andrews responded that not all are, it depends on which tasks they're interacting or running with. Mark Gross then asked how much work is done in background tasks. The Android developers said not much, but it can be intermittent, though they didn't have hard numbers.

Andrews then mentioned that the team really does want to better enforce the notion of foreground and background tasks. Even when background tasks are limited by cpusets and lower priorities, some still cause scheduling interference with foreground tasks. Stultz asked if pushing foreground tasks to `SCHED_RR` (the round-robin realtime scheduling class) would solve this, but Andrews noted that the team was really avoiding making tasks with non-deterministic runtimes run as round-robin.

Stultz then asked how the cpusets interacted with Android's use of cgroups. Kjos clarified that they help together. Android still uses cgroups to regulate task CPU usage, but the cpusets help pin tasks to processors, so the tasks don't otherwise fan out to the big CPUs. Batacharia noted that EAS should help with that problem.

As the session wrapped up, Kjos indicated that these types of systems will be an area of greater focus for his team in the coming year.

At that point, after over six hours, the microconference came to an end, and everyone quickly left for dinner and drinks.

[Thank you to all the presenters for their discussions, Karim Yaghmour for organizing and running the conference, and Rom Lemarchand for helping get so many of the Google Android team to attend.]
