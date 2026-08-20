---
title: WireGuarding the mainline
url: https://lwn.net/Articles/761939/
date: "August 6, 2018"
category: "Cryptography; Networking-Virtual private networks"
author: "By Jonathan Corbet August 6, 2018"
---

By **Jonathan Corbet**  
August 6, 2018

The [WireGuard](<https://www.wireguard.com/>) VPN tunnel has been under development — and attracting attention — for a few years now; LWN [ran a review](<https://lwn.net/Articles/748582/>) of it in March. While WireGuard can be found in a number of distribution repositories, it is not yet shipped with the mainline kernel because its author, Jason Donenfeld, hasn't gotten around to proposing it for upstreaming. That changed on July 31, when Donenfeld [posted WireGuard for review](<https://lwn.net/ml/linux-kernel/20180731191102.2434-1-Jason@zx2c4.com/>). Getting WireGuard itself into the mainline would probably not be all that hard; merging some of the support code it depends on could be another story, though. 

WireGuard implements a simple tunneling protocol allowing network traffic to be routed through a virtual private network provider. It has been developed with an eye toward smallness, ease of verification, and performance, rather than large numbers of features. It is, according to the patch posting, ""used by some massive companies pushing enormous amounts of traffic"". Some effort has gone into making WireGuard widely available, an effort that has helped to create a significant user community. But the ultimate way to make this kind of software widely available is to get it into everybody's kernel; that requires upstreaming. 

Thus far, the WireGuard code itself does not appear to be hugely controversial. There are plenty of little issues to deal with, including the inevitable (for networking code) demand that variable declarations be put into "reverse Christmas tree" order, along with a few technical issues that would appear to be easily resolved. Dealing with those things will likely take a couple of iterations, but the real fate of WireGuard is likely to be driven by [this "review"](<https://lwn.net/ml/linux-kernel/CA+55aFz5EWE9OTbzDoMfsY2ez04Qv9eg0KQhwKfyJY0vFvoD3g@mail.gmail.com/>) from Linus Torvalds: 

I see that Jason actually made the pull request to have wireguard included in the kernel. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Can I just once again state my love for it and hope it gets merged soon? Maybe the code isn't perfect, but I've skimmed it, and compared to the horrors that are OpenVPN and IPSec, it's a work of art. 

There is a potential sticking point, though. WireGuard, as one would expect, encrypts traffic between the ends of the tunnel, and endpoints perform cryptographic authentication of their peers. The kernel offers [an extensive cryptographic subsystem](<https://www.kernel.org/doc/html/latest/crypto/index.html>) that could be used to perform these operations, but Donenfeld turns out not to be a fan of that API. In an ["upstreaming roadmap"](<https://lwn.net/Articles/748584/>) posted last November, he stated his intent to ""embark on a multi-pronged effort to overhaul the crypto API"". In the end, he has created a completely new cryptographic subsystem for the kernel and based WireGuard on that rather than on the existing cryptographic code. 

Even without seeing this work, one can imagine that an effort to thrash up a longstanding core API would be a relatively hard sell. Donenfeld seemed to be trying to make things harder yet: his "Zinc" cryptography API was posted as a single, 24,000-line patch that was duly rejected by the mailing-list filters. It is [available in his repository](<https://git.kernel.org/pub/scm/linux/kernel/git/zx2c4/linux.git/commit/?h=zinc>), though, for those who want to take a look. The changelog describes the motivations for Zinc in the sort of, shall we say, highly self-assured language favored by security-oriented developers. It seems designed to raise enough hackles to prevent serious consideration of what is actually being proposed. 

In short, Donenfeld argues, the kernel's cryptographic API is far too complex and difficult to use. It is designed to work with acceleration hardware and allow sophisticated composition of operations, requiring users to do things like set up transform contexts and scatter/gather lists for the data to be encrypted. But most users simply want to perform a simple cryptographic operation and do not benefit from the complexity of the kernel's API, so developers tend to avoid it. What's needed, he says, is a different sort of cryptographic library: 

The kernel is in the business usually not of coming up with new uses of crypto, but rather implementing various constructions, which means it essentially needs a library of primitives, not a highly abstracted enterprise-ready pluggable system, with a few particular exceptions. 

Zinc follows this philosophy by providing a simple set of cryptographic algorithms that can be called without any elaborate setup rituals. A number of those algorithms duplicate functionality that already exists in the kernel's cryptographic layer. There is talk in the changelog about eventually reworking the in-kernel code to be based on Zinc, but that work has not been done at this time. 

While Zinc has not exactly been received with open arms, neither has it been rejected out of hand. Donenfeld does actually have a point when it comes to the complexity of the current cryptographic API. The [most detailed review](<https://lwn.net/ml/linux-kernel/20180801072246.GA15677@sol.localdomain>) so far has come from kernel cryptographic developer Eric Biggers who, after saying that he wanted to see WireGuard upstream, said that ""a few things are on the wrong track"" on the Zinc side. He would like to see the various algorithms split out and added separately, for example, making the patch set easier to review. He pointed out that Zinc cannot support hardware cryptographic accelerators, something that Donenfeld regards as a feature. All told, he seems to not be opposed to the overall concepts behind Zinc, but would like to see a number of changes in the implementation. 

Andy Lutomirski [was generally favorable](<https://lwn.net/ml/linux-kernel/CALCETrXD_VyoyW5C1U34o-ySJ4nFcO6PQ+ZNBVYxCpJ-f65CHw@mail.gmail.com/>) as well, noting that he has tried to carry out some similar changes to the cryptographic code in the past. Support for hardware accelerators should, he said, be built on top of Zinc; code needing that support could then use the more complex API that would be required, and the Zinc implementations could be used as fallbacks when acceleration is not available or practical to use. Lutomirski supported a number of Biggers's requests for changes. Meanwhile, Herbert Xu, the maintainer of the cryptographic subsystem, has stayed out of the discussion. 

Donenfeld has been generally receptive to the requests for changes, and has promised a new version of Zinc with many of the issues raised so far addressed. That will almost certainly not be the end of the discussion; that kind of deep surgery on a core kernel subsystem does not happen overnight. But if all of the participants in the conversation continue to be open to what the others are saying, the remaining kinks should be ironed out before too long, and WireGuard will finally have a path into the mainline.
