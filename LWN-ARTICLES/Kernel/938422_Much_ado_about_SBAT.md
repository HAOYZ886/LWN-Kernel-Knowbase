---
title: Much ado about SBAT
url: https://lwn.net/Articles/938422/
date: "July 20, 2023"
category: "Security-UEFI secure boot"
author: "By Jonathan Corbet July 20, 2023"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 20, 2023

Sometimes, the shortest patches lead to the longest threads; for a case in point, see [this three-line change](<https://lwn.net/ml/linux-kernel/20230711154449.1378385-1-eesposit@redhat.com/>) posted by Emanuele Giuseppe Esposito. The purpose of this change is to improve the security of locked-down systems by adding a "revocation number" to the kernel image. But, as the discussion revealed, both the cost and the value of this feature are seen differently across the kernel-development community. 

The patch in question adds three lines to the kernel's x86 makefile, the end result of which is to add a new ELF section to the kernel executable, called "`.sbat`", containing these two lines of text: 

```
sbat,1,SBAT Version,sbat,1,https://github.com/rhboot/shim/blob/main/SBAT.md
        linux,1,The Linux Developers,linux,$(KERNELVERSION),https://linux.org"
```

Where `$(KERNELVERSION)` is replaced with the current kernel version number. The first line describes the section as conforming to version 1 of the [UEFI Secure Boot Advanced Targeting (SBAT) specification](<https://github.com/rhboot/shim/blob/main/SBAT.md>). The second line is the one that matters; it identifies the executable component as "`linux`", with version number 1 (also called the "generation number"); the rest of the line is just documentation. 

The purpose of the SBAT entry is to give the bootloader a way to know whether the given executable is safe to run. Since we are dealing with secure boot, that executable should already be signed by a key known to the system, but it is possible that a security vulnerability has been found in that binary since it was signed. To protect against that possibility, the bootloader will contain a minimum acceptable generation number; if the generation found in the SBAT section is below that number, the bootloader will refuse to boot it. Whenever a vulnerability that will lead to a secure-boot compromise is fixed, the minimum generation will be incremented, with the effect of immediately blocking the loading of any vulnerable versions of the code. 

The SBAT idea is not new; it has been supported by the GRUB bootloader since the 2.06 release. The developers of [the unified kernel image concept](<https://0pointer.de/blog/brave-new-trusted-boot-world.html>) are implementing ways to load a kernel directly, without using a separate bootloader; as a part of this work, they want to add SBAT checks for the kernel image itself. The addition of SBAT version number to the build is a necessary part of this scheme. 

The secure boot mechanism already contains ways to prevent the loading of old, vulnerable images through explicit revocation. There is just one little problem: given the diversity of the software used in the Linux world — and the large number of releases of each — the revocation lists quickly grow larger than the low-level firmware implementing secure boot can handle. SBAT is an attempt to overcome that problem by creating a single number that can be used to determine whether a given image is safe to boot. 

#### Generation-number management

Much of the above was not really explained in the patch posting, which led to a number of questions on the list. Other signs of inattention (the inclusion of a linux.org URL, for example) also raised red flags. It is fair to say that this idea did not get the reception that its backers were hoping for. 

At its core, the disagreement was over the management of that version number: how often would it change, and who would be responsible for deciding when it should be changed? Greg Kroah-Hartman [suggested](<https://lwn.net/ml/linux-kernel/2023071229-dusk-repacking-da3a@gregkh/>) that the kernel's version number, which is already included in the image, could be used instead, but Luca Boccassi [replied](<https://lwn.net/ml/linux-kernel/CAMw=ZnSmZTBs+bJsQ_Y2CVO8K3OTuHOZDKW4cbxKpGbo4Vgs7Q@mail.gmail.com/>) that the version number would not work, since there are too many of them for the system to track. In other words, kernel-version numbers present the same problem that revocation lists do — the problem that SBAT was developed to work around. 

Kroah-Hartman later [asked](<https://lwn.net/ml/linux-kernel/2023071233-empirical-overturn-744c@gregkh/>) a number of questions about how the decision to update the generation number would be made, who would make it, who would change the number in the kernel, and so on. He suggested looking at past kernel history to come up with an idea of how many times that number would have changed over time. Boccassi's [answers](<https://lwn.net/ml/linux-kernel/CAMw=ZnRRviBNi_LK9VOSUV9PNHe3jurUcLfgLpbTOsp_eE4WyA@mail.gmail.com/>) were not seen as entirely satisfactory; he said that the decision would be made by: ""most likely those who understand the problem space"". He suggested that the kernel project makes ""3 releases a year"", and that generation-number changes would be no more frequent than that. Kroah-Hartman [replied](<https://lwn.net/ml/linux-kernel/2023071350-specked-botanist-6ba8@gregkh/>) that he does ""a release or two a week across multiple stable kernel versions"", rather more than three per year, and repeated his process questions. 

Boccassi called the stable releases ""irrelevant for the case at hand"" and said that the process questions didn't matter: ""the question here was about mechanism and storage. And it already works btw, it's just the kernel that's lagging behind, as usual"". Kroah-Hartman [was not moved](<https://lwn.net/ml/linux-kernel/2023071552-quilt-tranquil-b7bf@gregkh/>): 

> To think that "let's add a security canary to the kernel image" is anything other than a process issue shows a lack of understanding about exactly how the kernel is released, how the existing kernel security response team works, and who does any of this work. To ignore that means that there is no way in the world this can ever be accepted. 

Daniel P. Berrangé did [make an attempt](<https://lwn.net/ml/linux-kernel/ZLUIViihakhyPV1N@redhat.com/>) to address the process-related questions, saying that generation-number increments would be tied to CVE numbers and would likely be infrequent. He also acknowledged that there are some open questions with regard to how backports to the stable releases would be handled. Kroah-Hartman [responded](<https://lwn.net/ml/linux-kernel/2023071723-paging-despite-7094@gregkh/>) that most security-relevant kernel bugs are never assigned CVE numbers, and that he knows of many example of bugs that could be used to break secure boot. He also admonished: ""as the person running the stable releases, you BETTER be working with me to try to figure this all out"". 

One might not think that the management of a simple number would be so hard. But the question of when it should be incremented is not trivial. As Kroah-Hartman and others pointed out, the kernel project is fixing bugs that may have security implications almost every day. Nobody knows how often exactly, because monitoring the patch stream for possible security issues is a task that nobody has the resources to keep up with for any extended period of time. It is probably safe to say, though, that almost every mainline kernel release has at least one fix for a bug that could be used to attack secure boot. So perhaps the generation number would simply need to increment for each release. 

There is a worse problem, though, in that almost nobody runs mainline releases; instead, most users are running kernels derived from the stable updates. It is far from clear how the mainline generation-number updates should be backported to the stable releases, which happen much more frequently than mainline releases. Each stable release may have a subset of the fixes that were identified as needing generation-number increments in the mainline; how should the generation number be calculated in such cases? If a given fix is not applicable to a specific kernel release, should that number be incremented anyway — thus causing older binaries to fail to boot, even though they lack the vulnerability in question? 

#### Letting distributors do it

For these reasons and more, it was occasionally suggested that, if such a generation number is to be a part of a kernel build, it should be created and managed by the distributors who are building the kernels. As Ard Biesheuvel [put it](<https://lwn.net/ml/linux-kernel/ZK%2F9MlTh435FP5Ji@gambale.home/>): 

> Therefore, I don't think it makes sense for the upstream kernel source to carry a revocation index. It is ultimately up to the owner of the signing key to decide which value gets signed along with the image, and this is fundamentally part of the configure/build/release workflow. No distro builds and signs the upstream sources unmodified, so each signed release is a fork anyway, making an upstream revocation index almost meaningless. 

Boccassi's [reply](<https://lwn.net/ml/linux-kernel/CAMw=ZnRQ0qNg0hT27DWvfZ1DH3dLdu0U7zYN8HvJDx7NPVy_pQ@mail.gmail.com/>), after describing the linux-kernel list as ""an open sewer"", dismissed this idea as unworkable: 

> The 'owner of the signing key' is not good enough, because there are many of those - as you know, the kernel is signed by each distro. But the key here is that the revocation is _global_ (again: global means it applies to everyone using shim signed by 3rd party CA), so each distro storing their own id defeats the purpose of that. 

If this global number is not stored in the mainline kernel source, he said, somebody would have to maintain an external registry to somehow map generation numbers to points in the kernel's development history. 

Paolo Bonzini, though, [thought](<https://lwn.net/ml/linux-kernel/df7941b9-11ea-9abd-e070-4e9926aecdc3@redhat.com/>) that even a distributor-managed generation number is unworkable: ""I'm quite positive that a revocation index attached to the kernel image cannot really work as a concept, not even if it is managed by the distro"". That led to [another missive](<https://lwn.net/ml/linux-kernel/CAMw=ZnTSyeL4x1o_GVKNsZdCUe2D2H53quK-TSFhOkuONjC73A@mail.gmail.com/>) from Boccassi stating that the mechanism has been shown to work elsewhere, and that ""the kernel is not special in any way"": 

> The only thing that matters is if, given a bug, somebody either observed it being used as a secure boot bypass by bad actors in the wild, or was bothered enough to write down a self-contained, real and fully working proof of concept for secure boot bypass. If yes, then somebody will send the one-liner to bump the generation id, and a new sbat policy will be deployed. If no, then most likely nobody will care, and that's fine, and I expect that's what will happen most of the time. 

It is not clear that this approach will satisfy the developers who see the whole mechanism as a sort of security theater. 

As of this writing, the discussion appears to be at an impasse, with little mutual understanding between the participants. The proponents of the SBAT mechanism see a way of addressing their revocation problems that only needs an occasional one-line kernel patch to maintain. Longtime kernel developers, though, see a raft of unresolved process issues and strongly doubt that a single integer value can describe the security status of the huge variety of kernels in the wild. The kernel is more complicated that that, as is the security environment it operates in; any sort of global revocation mechanism may have to be as well.
