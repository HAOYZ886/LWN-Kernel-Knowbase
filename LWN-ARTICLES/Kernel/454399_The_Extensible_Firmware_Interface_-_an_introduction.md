---
title: "The Extensible Firmware Interface - an introduction"
url: https://lwn.net/Articles/454399/
date: "August 9, 2011"
category: Externsible firmware interface
author: "August 9, 2011 This article was contributed by Matthew Garrett"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

August 9, 2011

This article was contributed by Matthew Garrett

In the beginning was the BIOS. 

Actually, that's not true. Depending on where you start from, there was either some toggle switches used to enter enough code to start booting from something useful, a ROM that dumped you straight into a language interpreter or a ROM that was just barely capable of reading a file from tape or disk and going on from there. CP/M was usually one of the latter, jumping to media that contained some hardware-specific code and a relatively hardware-agnostic OS. The hardware-specific code handled receiving and sending data, resulting in it being called the "Basic Input/Output System." BIOS was born. 

When IBM designed the PC they made a decision that probably seemed inconsequential at the time but would end up shaping the entire PC industry. Rather than leaving the BIOS on the boot media, they tied it to the initial bootstrapping code and put it in ROM. Within a couple of years vendors were shipping machines with reverse engineered BIOS reimplementations and the PC clone market had come into existence. 

There's very little beauty associated with the BIOS, but what it had in its favor was functional hardware abstraction. It was possible to write a fairly functional operating system using only the interfaces provided by the system and video BIOSes, which meant that vendors could modify system components and still ship unmodified install media. Prices nosedived and the PC became almost ubiquitous. 

The BIOS grew along with all of this. Various arbitrary limits were gradually removed or at least papered over. We gained interfaces for telling us how much RAM the system had above 64MB. We gained support for increasingly large drives. Network booting became possible. But limits remained. 

The one that eventually cemented the argument for moving away from the traditional BIOS turned out to be a very old problem. Hard drives still typically have 512 byte sectors, and the MBR partition table used by BIOSes stores sectors in 32-bit variables. Partitions above 2TB? Not really happening. And while in the past this would have been an excuse to standardize on another BIOS extension, the world had changed. The legacy BIOS had lasted for around 30 years without ever having a full specification. The modern world wanted standards, compliance tests and management capabilities. Something clearly had to be done. 

And so for the want of a new partition table standard, EFI arrived in the PC world. 

#### Expedient Firmware Innovation

[1] Intel's other stated objection to Open Firmware was that it had its own device tree which would have duplicated the ACPI device tree that was going to be present in IA64 systems. One of the outcomes of the OLPC project was an Open Firmware implementation that glued the ACPI device tree into the Open Firmware one without anyone dying in the process, while meanwhile EFI ended up allowing you to specify devices in either the ACPI device tree or through a runtime enumerated hardware path. The jokes would write themselves if they weren't too busy crying. 

[2] To be fair to Intel, choosing to have drivers be written in C rather than Forth probably did make EFI more attractive to third party developers than Open Firmware 

Intel had at least 99 problems in 1998, and IA64 was certainly one of them. IA64 was supposed to be a break from the PC compatible market, and so it made sense for it to have a new firmware implementation. The 90s had already seen several attempts at producing cross-platform legacy-free firmware designs with the most notable probably being the ARC standard that appeared on various MIPS and Alpha platforms and Open Firmware, common on PowerPC and SPARCs. ARC mandated the presence of certain hardware components and lacked any real process for extending the specification, so got passed over. Open Firmware was more attractive but had a very limited third party developer community[1], so the choice was made to start from scratch in the hope that a third party developer community would be along eventually[2]. This was the Intel Boot Initiative, something that would eventually grow into EFI. 

EFI is intended to fulfill the same role as the old PC BIOS. It's a pile of code that initializes the hardware and then provides a consistent and fairly abstracted view of the hardware to the operating system. It's enough to get your bootloader running and, then, for that bootloader to find the rest of your OS. It's a specification that's 2,210 pages long and still depends on the additional 727 pages of the ACPI spec and numerous ancillary EFI specs. It's a standard for the future that doesn't understand surrogate pairs and so can never implement full Unicode support. It has a scripting environment that looks more like DOS than you'd have believed possible. It's built on top of a platform-independent open source core that's already something like three times the size of a typical BIOS source tree. It's the future of getting anything to run on your PC. This is its story. 

#### Eminently Forgettable Irritant

[3] The latest versions of EFI allow for a pre-PEI phase that verifies that the EFI code hasn't been modified. We heard you like layers. 

[4] Those of you paying attention have probably noticed that the PEI sounds awfully like a BIOS, EFI sounds awfully like an OS and bootloaders sound awfully like applications. There's nothing standing between EFI and EMACS except a C library and a port of readline. This probably just goes to show something, but I'm sure I don't know what. 

The theory behind EFI is simple. At the lowest level[3] is the Pre-EFI Initialization (PEI) code, whose job it is to handle setting up the low-level hardware such as the memory controller. As the entry point to the firmware, the PEI layer also handles the first stages of resume from S3 sleep. PEI then transfers control to the Driver Execution Environment (DXE) and plays no further part in the running system. 

The DXE layer is what's mostly thought of as EFI. It's a hardware-agnostic core capable of loading drivers from the Firmware Volume (effectively a filesystem in flash), providing a standardized set of interfaces to everything that runs on top of it. From here it's a short step to a bootloader and UI, and then you're off out of EFI and you don't need to care any more[4]. 

The PEI is mostly uninteresting. It's the chipset-level secret sauce that knows how to turn a system without working RAM into a system with working RAM, which is a fine and worthy achievement but not typically something an OS needs to care about. It'll bring your memory out of self refresh and jump to the resume vector when you're coming out of S3. Beyond that? It's an implementation detail. Let's ignore it. 

The DXE is where things get interesting. This is the layer that presents the interface embodied in the EFI specification. Devices with bound drivers are represented by handles, and each handle may implement any number of protocols. Protocols are uniquely identified with a GUID. There's a `LocateHandle()` call that gives you a reference to all handles that implement a given protocol, but how do you make the `LocateHandle()` call in the first place? 

This turns out to be far easier than it could be. Each EFI protocol is represented by a table (ie, a structure) of data and function pointers. There's a couple of special tables which represent boot services (ie, calls that can be made while you're still in DXE) and runtime services (ie, calls that can be made once you've transitioned to the OS), and in turn these are contained within a global system table. The system table is passed to the main function of any EFI application, and walking it to find the boot services table then gives a pointer to the `LocateHandle()` function. Voila. 

So you're an EFI bootloader and you want to print something on the screen. This is made even easier by the presence of basic console io functions in the global EFI system table, avoiding the need to search for an appropriate protocol. A "Hello World" function would look something like this: 

```
#include <efi.h>
        #include <efilib.h>
    
        EFI_STATUS
        efi_main (EFI_HANDLE image, EFI_SYSTEM_TABLE *systab)
        {
            SIMPLE_TEXT_OUTPUT_INTERFACE *conout;
    
            conout = systab->ConOut;
            uefi_call_wrapper(conout->OutputString, 2, conout, L"Hello World!\n\r");
    
            return EFI_SUCCESS;
        }
```

In comparison, graphics require slightly more effort: 

```
#include <efi.h>
        #include <efilib.h>
    
        extern EFI_GUID GraphicsOutputProtocol;
    
        EFI_STATUS
        efi_main (EFI_HANDLE image, EFI_SYSTEM_TABLE *systab)
        {
            EFI_GRAPHICS_OUTPUT_PROTOCOL *gop;
    	EFI_GRAPHICS_OUTPUT_MODE_INFORMATION *info;
    	UINTN SizeOfInfo;
    
    	uefi_call_wrapper(BS->LocateProtocol, 3, &GraphicsOutputProtocol,
    	                  NULL, &gop);
    	uefi_call_wrapper(gop->QueryMode, 4, gop, 0, &SizeOfInfo, &info);
    
    	Print(L"Mode 0 is running at %dx%d\n", info->HorizontalResolution,
    	      info->VerticalResolution);
    
    	return 0;
        }
```

[5] Well, except that things are obviously more complicated. It's possible for multiple device handles to implement a single protocol, so you also need to work out whether you're speaking to the right one. That can end up being trickier than you'd like it to be. 

Here we've asked the firmware for the first instance of a device implementing the Graphics Output Protocol. That gives us a table of pointers to graphics related functionality, and we're free to call them as we please.[5] 

#### Extremely Frustrating Issues

So far it all sounds straightforward from the bootloader perspective. But EFI is full of surprising complexity and frustrating corner cases, and so (unsurprisingly) attempting to work on any of this rapidly leads to confusion, anger and a hangover. We'll explore more of the problems in the next part of this article.
