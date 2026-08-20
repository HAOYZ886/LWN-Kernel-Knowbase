---
title: Systemd catches up with bind events
url: https://lwn.net/Articles/837033/
date: "November 13, 2020"
category: "Development model-User-space ABI; udev"
author: "By Jonathan Corbet November 13, 2020"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 13, 2020

The kernel project has a strong focus on not breaking user-space applications; if something works with a given kernel release, it should continue to work with subsequent releases. So it may be discouraging to read the lengthy exposition on an apparent user-space API break in [the announcement](<https://lwn.net/Articles/837034/>) for the systemd 247-rc2 release. Changes to udev configuration files will be needed to keep systems working, but the systemd project claims that it ""is not [the] fault of systemd or udev, but caused by an incompatible kernel change that happened back in Linux 4.12"". It seems like an appropriate time to look at what happened, how administrators need to respond, and whether anything can be done to avoid this kind of thing from happening again. 

Modern computers tend to be highly dynamic, with devices (of both the physical and virtual variety) appearing and disappearing while the system is running. The kernel handles the low-level details with regard to these device events, but it is up to user space to take care of the rest. For that to happen, user space needs to know when something has changed with the system's configuration. 

To that end, events are emitted to user space from deep within the kernel's driver-core subsystem whenever something changes; for example, plugging in a USB device will result in the creation of one or more `ADD` events to tell user space that the new device is available. The `udev` daemon is charged with responding to these events according to a set of rules; it can create device nodes, set permissions, notify other user-space components, and more, all in response to properties attached to events by matching rules. The set of possible events is relatively small and does not change often. 

#### Breaking systemd

In July 2017, though, Dmitry Torokhov [added two new event types](<https://git.kernel.org/linus/1455cf8dbfd0>) called `BIND` and `UNBIND`. They are meant to allow user space to handle devices that need help before they can become fully functional — those that need a firmware load, for example. For drivers that support the new mechanism, a `BIND` event for a device will follow the `ADD` event once the device is ready to operate. This change was a part of the 4.14 kernel release in November 2017 (not 4.12 as stated in the systemd announcement). 

Later that same month, [a bug report](<https://bugs.kde.org/show_bug.cgi?id=387454>) landed in the KDE bug tracker; this was perhaps the first case where somebody noticed a problem related to the new events. That report only [made it to the kernel lists](<https://lwn.net/ml/linux-kernel/20181223164954.hib4lbchftspidsd@suse.de/>) at the end of 2018, though — over one year later. By then, 4.14 had been made into a long-term support kernel and shipped by distributors, with relatively few complaints from users. Indeed, Greg Kroah-Hartman [was mystified](<https://lwn.net/ml/linux-kernel/20181224091229.GA26796@kroah.com/>) as to why problems were turning up a year later. That turned out to be a change to systemd that caused it to propagate the new events. 

Specifically, the problem would appear to originate in the way that udev (which is a part of the systemd project) attaches tags to events. These tags, which are set and used by udev rules, control how user space will set up the new device. There is an assumption built in that there will only be a single event to announce the existence of a new device, so attaching tags to that event is sufficient. When the second event (the `BIND` event) shows up, the device state is reset and those tags are forgotten, leading to the associated device not being set up properly. 

As a short-term "fix", systemd was [patched](<https://github.com/systemd/systemd/commit/56c886dc7e>) to simply ignore the new events. That caused things to work as they did before, at the cost of hiding those events entirely. That was never a long-term solution; the new events were added for a reason and some devices need them for proper setup. So a better solution had to be found for the longer term; that solution has two aspects, one of which may be disruptive for users who have created their own udev rules. 

#### Fixing systemd

The first piece is a reworking of the "tag" mechanism provided by udev. Tags are special properties that can be attached, then matched in subsequent rules or consumed by user space. Rather than attaching tags to events, as has been done until now, udev attaches them to devices, so tags added in response to an `ADD` event will still be there for the `BIND` event as well. For cases where rules need to respond only to tags added to the current event, a new `CURRENT_TAGS` property lists only those tags; it thus holds the value that the `TAGS` property held in previous releases. 

The other part, though, is a change that must be applied to a number of udev rule sets. Consider, for example, this snippet taken from a randomly chosen rules file (`10-dm-disk.rules` in particular) on a Fedora 32 system: 

```
# "add" event is processed on coldplug only!
        ACTION!="add|change", GOTO="dm_end"
```

The `ACTION` line causes the entire file to be skipped for anything other than `ADD` or `CHANGE` events; in particular, that is what will happen with `BIND` events. That will cause properties associated with those events to be lost — and the device in question to be set up improperly (if at all). The fix is to change that line to read: 

```
ACTION=="remove", GOTO="dm_end"
```

That causes the rules to be skipped (and their associated state forgotten) only when the device is removed from the system. 

The problem here is that these rules were written under the assumption that no new event types would be added, so anything that wasn't recognized as adding or modifying a device could be ignored. There is, evidently, a certain amount of code that runs in response to device events that has a similar problem. What this shows is, in effect, a sort of [protocol ossification](<https://en.wikipedia.org/wiki/Protocol_ossification>) effect that has made it much harder to add event types to the API provided by the kernel. Indeed, in 2018, Torokhov [remarked](<https://lwn.net/ml/linux-kernel/20181224173427.GA254245@dtor-ws/>): 

Well, it appears that we can no longer extend uevent interface with new types of uevents, at least not until we go and fix up all udev-derivatives and give some time for things to settle. 

At the time, there was discussion of possibly reverting the change, causing the new events to disappear. But that approach had the potential to create regressions of its own, as some systems may well have depended on getting those events; the kernel release adding them was a year old by that point, after all. There was also discussion of adding some sort of knob to enable or disable the creation of `BIND` and `UNBIND` events, but that never came to pass. Instead, Torokhov [described](<https://lwn.net/ml/linux-kernel/20181224092257.GB122208@dtor-ws/>) the work in the systemd project to make the changes described above, and Kroah-Hartman [responded](<https://lwn.net/ml/linux-kernel/20181224093458.GA28717@kroah.com/>): ""So all should be good"". 

#### A regression?

With luck, all _will_ be good, but it has come at the cost of some work within the systemd community over the last two years; the systemd developers have made their displeasure known: 

We are very sorry for this breakage and the requirement to update packages using these interfaces. We'd again like to underline that this is not caused by systemd/udev changes, but result of a kernel behaviour change. 

Was this a violation of the kernel's "no regressions" rule? The answer must almost certainly be "yes"; code that worked with 4.13 no longer worked with 4.14. What should have been done about it is a bit less clear. Had the issue been reported to the kernel community more quickly, it might have been possible to revert and redesign the change; after it had been deployed for a year, though, that was not a simple option. One could argue that the kernel community should have found some other way to fix the regression; the systemd 247-rc2 announcement tries to make that case. But once Torokhov posted that the problem was being addressed on the systemd side, there was no longer any pressure to do that. 

Perhaps the real lesson here is that the community would be better served by closer relations between the kernel project and projects managing low-level utilities like systemd. Those relations have been somewhat strained at times, and there are not a lot of places where cooperative, cross-project discussions can take place. The presence of systemd developers at events like the Linux Plumbers Conference is limited at best, and those developers — not without reason — do not find the kernel mailing lists to be an entirely welcoming place. We are all working on the same system, though, and we would probably have an easier time of it if we could talk things through a bit more.
