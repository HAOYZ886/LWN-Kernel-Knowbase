---
title: "An alternative device-tree source language"
url: https://lwn.net/Articles/730217/
date: "August 9, 2017"
category: Device tree
author: "By Jonathan Corbet August 9, 2017"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 9, 2017

Device trees have become, in a relatively short time, the preferred way to inform the kernel of the available hardware on systems where that hardware is not discoverable — most ARM systems, among others. In short, a device tree is a textual description of a system's hardware that is compiled to a simple binary format and passed to the kernel by the bootloader. The source format for device trees has been established for a long time — longer than Linux has been using it. Perhaps it's time for a change, but a proposal for a new device-tree source format has generated a fair amount of controversy in the small corner of the community that concerns itself with such things. 

It may be hard to believe that, as recently as 2011, the use of device trees for the ARM architecture was controversial. Over time, though, device-tree proponents won out; ARM board files are mostly gone and unlamented, and the kernel source tree contains 2400 or so device-tree files. The [source format for device trees](<http://elinux.org/Device_Tree_Usage>), though, was never really discussed; the kernel community simply went with the format that was laid out in OpenFirmware many years ago. It was well defined, and there didn't seem to be any real reason for anybody to seek to change it. 

As a simple example, consider this fragment of the [Beaglebone Black device tree](<https://lwn.net/Articles/730224/>): 

```
leds {
    	pinctrl-names = "default";
    	pinctrl-0 = <&user_leds_s0>;
    	compatible = "gpio-leds";
    
    	led2 {
    	    label = "beaglebone:green:heartbeat";
    	    gpios = <&gpio1 21 GPIO_ACTIVE_HIGH>;
    	    linux,default-trigger = "heartbeat";
    	    default-state = "off";
    	};
        };
```

This fragment describes the green "heartbeat" LED; it indicates which GPIO line it is connected to and more. A device-tree source file for a complex system-on-chip contains a great deal of text like the above. This device-tree source (DTS) file will be fed to the compiler (`dtc`) to generate a binary "device-tree blob" (or DTB) file. That file, in turn, will be handed to the kernel when the system boots. 

Pantelis Antoniou recently showed up on the device-tree list with [a proposal](<https://lwn.net/Articles/729156/>) for a new device-tree compiler called `yamldt`. It produces the same DTB files as `dtc`, but the source format is different. Rather than go with the classic DTS format, `yamldt` deals with device trees expressed in the [YAML](<http://www.yaml.org/>) format. So, the above fragment, in the new scheme, would look like this: 

```
leds:
            pinctrl-names: "default"
            pinctrl-0: *user_leds_s0
            compatible: "gpio-leds"
            led2:
                label: "beaglebone:green:heartbeat"
                gpios: [ *gpio1, 21, GPIO_ACTIVE_HIGH ]
                linux,default-trigger: "heartbeat"
                default-state: "off"
```

Antoniou would appear to have a number of reasons for wanting to make this change. The DTS language is used only for device trees; few developers are familiar with it. YAML, instead, is ubiquitous and well understood. DTS requires its own parser, while YAML parsers exist for almost every language one can imagine. Text editors tend to have built-in modes for editing YAML files. Simplified YAML can even be easily understood by low-level code like bootloaders; that, in turn, makes it possible (and relatively easy) for the bootloader to edit the device tree on the fly at system boot time. This editing is needed to support [device-tree overlays](<https://lwn.net/Articles/616859/>) for dynamic hardware. ""I feel that the reliance on DTS has been holding progress back in expressing modern hardware"", he [said](<https://lwn.net/Articles/730226/>); switching to a different source language is his way of addressing that problem. 

Another important issue, though, is validation of device trees. There are, in a sense, three components to any device-tree implementation. The device-tree "bindings" are a specification of how device trees are to be expressed; there are over 1000 binding descriptions in the kernel tree. For example, [Documentation/devicetree/bindings/leds/common.txt](<https://lwn.net/Articles/730227/>) specifies how all LED devices should be described in a DTS file. The real arbiter of what is correct, though, is the code in drivers that interprets device trees; what happens there may or may not match what the bindings say. Then there are the actual DTS files which may not properly match either the bindings or the driver code. There would be, as Tom Rini [pointed out](<https://lwn.net/Articles/730364/>), real value in a tool that could validate a DTS file against the relevant bindings and, maybe someday, also help with the implementation of device-tree consumer code in the kernel. 

Using YAML for both bindings and device-tree source files holds out the possibility of that kind of validation. Each type of device-tree node can be described in terms of the fields it may (and must) have and the data types for each; the compiler (or one of the existing YAML schema-checking tools) can then ensure that any source file follows the rules. This kind of validation has not yet been implemented, and it will not be an easy job since the bindings files now are not in any sort of machine-readable format. But the possibility is there. 

Even so, Antoniou's work would appear to be facing some fairly strong headwinds. One of the kernel's device-tree maintainers, Frank Rowand, [replied](<https://lwn.net/Articles/730348/>): 

Keep in mind one of the reasons Linus says he is very direct is to avoid leading a developer on, so that they don't waste a lot of time trying to resolve the maintainer's issues instead of realizing that the maintainer is saying "no". Please read my current answer as being "no, not likely to ever be accepted", not "no, not in the current form". 

Rowand [softened](<https://lwn.net/Articles/730349/>) his position a bit after learning more about the work, but he was not alone in expressing skepticism. Perhaps that is not surprising; kernel developers who come in proposing major changes often get a conservative response at the beginning. 

Some of the developers involved appear to fear the prospect of converting vast numbers of DTS files into a new format. That task is not hard — Antoniou has tested his tools by converting the entire set of in-kernel DTS files to YAML, then verifying that they compile to the same DTB files — but it does involve a lot of churn. Antoniou is not actually proposing such a change, though; he seems to see YAML as an alternative format, rather than a replacement for DTS. But, as Rob Herring (another kernel device-tree maintainer) [put it](<https://lwn.net/Articles/730350/>): ""If YAML solves a bunch of problems, then of course we'd want to convert DTS files at some point"". 

David Gibson, the `dtc` maintainer, had [a number of concerns](<https://lwn.net/Articles/730351/>), though he did concede that, had YAML been well established when `dtc` was written, it might have been chosen rather than DTS. But he pointed out that YAML is a more expressive language than DTS; in particular, it can express things that cannot be rendered into DTB in any straightforward way. That might, he suggests, encourage developers to write device trees that cannot be properly compiled. He also, like some others, suggested that the YAML source format does not help much with validation. Bindings could be put into YAML and used to validate device trees in the current DTS format. Antoniou naturally [disagreed](<https://lwn.net/Articles/730354/>), saying that the ability to use and track type information in YAML device-tree files will be an important part of a future validation mechanism. 

As the conversation wound down, Grant Likely [said](<https://lwn.net/Articles/730355/>) that the important contribution in this work may be the definition of a data model for device-tree data. That model needs to exist before any sort of formal validation can be done. In the end, that may be the part of this proposal that has the most influence going forward, though it is hard to tell for sure, since this conversation didn't reach any firm conclusions. Like the ARM device-tree conversion itself, this looks like a topic that will need several iterations on the mailing lists before some sort of consensus emerges.
