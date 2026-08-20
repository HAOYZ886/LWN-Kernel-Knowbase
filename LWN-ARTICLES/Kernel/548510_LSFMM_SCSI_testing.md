---
title: "LSFMM: SCSI testing"
url: https://lwn.net/Articles/548510/
date: "April 25, 2013"
category: SCSI
author: "By Jonathan Corbet April 25, 2013 LSFMM Summit 2013"
---

By **Jonathan Corbet**  
April 25, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

The Linux kernel deals extensively with SCSI devices, many of which have interpretations of the SCSI standard that can accurately be described as "interesting." That leaves developers on both the kernel and device sides of the bus looking for ways to verify that a specific hardware combination is working correctly. Lee Duncan led an LSFMM 2013 storage-track discussion on how SCSI testing might be improved. 

Once upon a time, Lee was using a set of Python scripts aimed at testing specific SCSI features, but then he stumbled across Ronnie Sahlberg's [libiscsi library](<https://github.com/sahlberg/libiscsi>), which has a "very handy" test suite built into it. Those tests cover a wide range of behavior, including things like response to malformed SCSI commands and various "edge cases." This kind of testing is useful, he said, to help determine whether a SCSI target is working correctly. Lots of SCSI targets are implemented in software and change frequently, so this is a question that needs to be answered often. His question was: is it possible to put this kind of testing infrastructure into the kernel's SCSI layer, or, failing that, to at least make this kind of testing easier? 

There was some discussion about whether SCSI 2 or SCSI 3 testing would be more useful. James Bottomley stepped in to "set expectations," saying that a lot of devices lie about what version of the SCSI standard they support. Almost every USB device claims to support SCSI 2, for example, though few of them do, and a lot of SCSI 2 devices fraudulently claim SCSI 3 compliance. So the question, he said, is somewhat irrelevant; in the real world, the lines between the various standards are blurry at best. James added that there are many heuristics in the SCSI stack which are keyed off the SCSI version. Martin Petersen said that it would be nice if the test suite could replicate those heuristics. 

There was wide agreement that a test suite would be useful; James said that device manufacturers have asked for such a thing and said they would use it if it were available. So the discussion turned to the form that this test suite would take. James said at the outset that it could not be built on top of [Open-iSCSI](<http://www.open-iscsi.org/>), because that would make it impossible to test the initiator side of the bus. Instead, the suite should attach to the SG (SCSI generic) interface and send commands via that path. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The set of commands sent to the device would need to be thought out a bit; as James noted, there is no desire to actually destroy the device being tested. So the suite would send "sane" commands and verify that they work properly. Martin added that there is value in hitting a device with random commands, but that's not what vendors are usually looking to test. 

James said that there are other things that would be nice to test but, due to lies from the device, are hard to test. The classic example is a device that includes a writeback cache to improve performance, but fails to implement the commands associated with caching control. Such devices have a high risk of losing data. This is a hard case to catch, though sometimes that can be done through repeated write-and-power-off cycles. There was also talk of testing devices with multiple paths to ensure that commands issued in parallel are executed properly. 

The session ended with a set of tasks to be carried out, starting with the need to come up with a list of commands that should be exercised by the test suite; these will then be built into a user-space suite. 

[Thanks to Elena Zannoni, whose extensive notes made this writeup possible.]
