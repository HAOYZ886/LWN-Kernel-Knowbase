---
title: Attestation for kernel patches
url: https://lwn.net/Articles/813646/
date: "March 2, 2020"
category: "Development tools; Security-Patch verification"
author: "By Jonathan Corbet March 2, 2020"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 2, 2020

The kernel development process is based on trust at many levels — trust in developers, but also in the infrastructure that supports the community. In some cases, that trust may not be entirely deserved; most of us have long since learned not to trust much of anything that shows up in email, for example, but developers still generally trust that emailed patches will be what they appear to be. In his ongoing effort to bring more security to kernel development, Konstantin Ryabitsev has proposed [a patch attestation scheme](<https://lwn.net/ml/workflows/20200226172502.q3fl67ealxsonfgp@chatter.i7.local/>) that could help subsystem maintainers verify the provenance of the patches showing up in their mailboxes. 

One might wonder why this work is needed at all, given that email attestation has been widely available for almost exactly as long as the kernel has existed; Phil Zimmermann first released PGP in 1991. PGP (and its successor, GnuPG) have always been painful to use, though, even before considering their interference with patches and the review process in particular; PGP-signed mail can require escaping characters or be mangled by mail transfer agents. It is safe to say that nobody bothers checking the few PGP signatures that exist on patches sent via email. 

Ryabitsev's goal is to make attestation easy enough that even busy kernel developers will be willing to add it to their workflow. The scheme he has come up with is, for now, meant for integration with processes that involve using `git send-email` to send out a set of patches, though it is not tightly tied to that workflow. A developer can add attestation to their process by creating a directory full of patches and sending them out via `git send-email` in the usual manner; attestation is then done as a separate step, involving an additional email message. 

In particular, the developer will run the `attest-patches` tool found in Ryabitsev's [korg-helpers repository](<https://git.kernel.org/pub/scm/linux/kernel/git/mricon/korg-helpers.git>). It will look at each patch and split it into three components: 

  * Some patch metadata: specifically the author's name and email address, along with the subject line. 
  * The commit message. 
  * The patch itself. 

The tool will use `sha256sum` to create a separate SHA-256 checksum for each of the three components. The three checksums are then joined, in an abbreviated form, to create a sort of unique ID for the patch that looks like: 

```
2a02abe0-215cf3f1-2acb5798
```

The `attest-patches` tool creates a file containing this "attestation ID", along with the full checksums for all three components: 

```
2a02abe0-215cf3f1-2acb5798:
          i: 2a02abe02216f626105622aee2f26ab10c155b6442e23441d90fc5fe4071b86e
          m: 215cf3f133478917ad147a6eda1010a9c4bba1846e7dd35295e9a0081559e9b0
          p: 2acb5798c366f97501f8feacb873327bac161951ce83e90f04bbcde32e993865
```

A block like this is generated for each patch given to `attest-patches`. The result happens to be a file in the YAML format, but one can live in ignorance of that fact without ill effect. The file is then passed to GnuPG for signing. The final step is to email this file to `signatures@kernel.org`, where it will appear on a public mailing list; `attest-patches` can perform this step automatically. 

On the receiving end, a reviewer or subsystem maintainer runs [`get-lore-mbox`](<https://lwn.net/Articles/811528/>) with the `-aA` options; `-A` does not actually exist yet but one assumes it will appear shortly. As the tool creates a mailbox file suitable for feeding to `git am`, it will verify the attestation for each of the patches it finds. That is done by generating its own attestation ID for each patch, then using that ID to search for messages on the `signatures` mailing list. If any messages are found, the full checksum for each of the three patch components is checked. The GPG signature in the file is also checked, of course. 

If the checks pass — meaning that an applicable signature message exists, the checksums match the patches in question, and the message is signed by a developer known to the recipient — then `get-lore-mbox` will create the requested mailbox file, adding a couple of tags to each patch describing the attestation that was found. Otherwise the tool will abort after describing where things went wrong. 

A test run of the system has already been done; Kees Cook generated [an attestation message](<https://lore.kernel.org/signatures/202002251425.E7847687B@keescook/>) for [this patch series](<https://lwn.net/ml/kernel-hardening/20200225051307.6401-1-keescook@chromium.org/>). He [said](<https://lwn.net/ml/workflows/202002260938.BFA7FA03@keescook/>) that this mechanism would be ""utterly trivial"" to add to his normal patch-generation workflow. 

Jason Donenfeld, instead, [was unconvinced](<https://lwn.net/ml/workflows/20200227041144.GA36493@zx2c4.com/>) of the value of this infrastructure. He argued that ""maintainers should be reading commits as they come in on the mailing list"" and that attestation would make the contribution process harder. He asked: ""is the lack of signatures on email patches a real problem we're facing?"" 

Ryabitsev [responded](<https://lwn.net/ml/workflows/20200227142935.4ulyjoodgyeu4uoz@chatter.i7.local/>) that he saw this mechanism as addressing two specific threats: 

  * An ""overworked, tired maintainer"" may be tempted to perform cursory reviews of patches from trusted developers; attestation at least lets them know that those patches actually came from their alleged author. 
  * Maintainers might diligently review patches arriving in email, then use a tool like `get-lore-mbox` to fetch those patches for easy application. If lore.kernel.org has been compromised, it could return a modified form of those patches and the maintainer may well never notice. Once again, attestation should block any such attack. 

He ended with a hope that the process he has developed is easy enough that developers will actually use it. 

Whether that will actually happen remains to be seen. The use of signed tags on pull requests is still far from universal, despite the fact that they, too, are easy to generate and Linus Torvalds requires them for repositories not hosted on kernel.org. Based on past discussions, it seems unlikely that Torvalds will require attestation for emailed patches. So if patch attestation is to become widespread in the kernel community, it will be as a result of lower-level maintainers deciding that it makes sense. Of course, a successful attack could change attitudes quickly.
