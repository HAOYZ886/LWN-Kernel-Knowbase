---
title: "Facing the Git commit-ID collision catastrophe"
url: https://lwn.net/Articles/1001526/
date: "December 13, 2024"
category: "Development tools-Git"
author: "By Jonathan Corbet December 13, 2024"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 13, 2024

Commits in the Git source-code management system are identified by the SHA-1 hash of their contents — though the specific hash [may change someday](<https://lwn.net/Articles/898522/>). The full hash is a 160-bit quantity, normally written as a 40-character hexadecimal string. While those strings are convenient for computers to work with, humans find them to be a bit unwieldy, so it is common to abbreviate the hash values to shorter strings. Geert Uytterhoeven recently [proposed](<https://lwn.net/ml/all/cover.1733421037.git.geert+renesas@glider.be>) increasing the length of those abbreviated hashes as used in the kernel community, but the problem he was working to solve may not be as urgent as it seems. 

A hash, of course, is not the same as the data it was calculated from; whenever hashes are used to represent data, there is always the possibility of a collision — when two distinct sets of data generate the same hash value. A 160-bit hash space is large enough that the risk of accidental collisions is essentially zero; the risk of intentional (malicious) collisions is higher, but is still not something that most people worry about — for now. The hash space is large enough that even a relatively small portion of the hash value is still enough to uniquely identify a value. In a small Git repository, a 24-bit (six-digit) hash may suffice; as a repository grows, the number of digits required to unambiguously identify a commit will grow. In all cases, though, the shorter commit IDs are much easier for humans to deal with, and are almost universally used. 

The kernel has, for some years now, used a twelve-character (48-bit) hash value in most places where a commit ID is needed. That is the norm for citing commits within changelogs (in Fixes tags, for example), and in email discussions as well. Uytterhoeven expressed a concern that, given the growth of the kernel repository, soon twelve digits will not be enough: ""the Birthday Paradox states that collisions of 12-character commit IDs are imminent"". He suggested raising the number of digits used to identify commits in the kernel repository to 16 to head off this possibility. 

Linus Torvalds, though, [made it clear](<https://lwn.net/ml/all/CAHk-=wiwAz3UgPOWK3RdGXDnTRHcwVbxpuxCQt_0SoAJC-oGXQ@mail.gmail.com>) that he did not support this change, for a couple of reasons. The first of those was that, while Git uses hashes to identify the objects in a repository, those objects are not all commits. There are three core object types in Git: blobs, trees, and commits. Blobs hold the actual data that the repository is managing — one blob holds the contents of one file at a given point in the history. Tree objects hold a list of blobs and their places in the filesystem hierarchy; they indicate which files were present in a given revision, and how they were laid out. If the only change between a pair of revisions is the renaming of a single file, the associated tree objects will differ only in that file's name; both will refer to the same blob object for the file's contents. Finally, a commit contains references to a number of objects (the previous commit(s) and the tree) along with metadata like the commit author, date, changelog, and so on. 

Torvalds's point was that commits only make up about 1/8 of the total objects in the repository. Even if two objects turn up with the same (shortened) hash, one of those objects is highly likely not to be a commit. Since humans rarely (never, in truth) traffic in blob or tree hashes, any collisions with those hashes are not a problem; it will be clear which one the human was referring to. When dealing with just the commit space, the problem of ambiguous abbreviations appears to be further away: 

> My tree is at about 1.3M commits, so we're basically an order of magnitude off the point where collisions start being an issue wrt commit IDs. 

When just looking at commit IDs, he said, there are no collisions when ten-digit abbreviations are used, so twelve seems safe for a while yet. Especially given that, as Torvalds pointed out, the current state was reached after nearly 20 years of use of Git within the kernel project. It will take a fair while yet to close that order-of-magnitude buffer that the kernel still has. 

Torvalds's other point, though, was that humans should not normally be quoting abbreviated hashes in isolation anyway. Within the kernel community, there is a strong expectation that commit IDs will be accompanied by the short-form version of the changelog. So rather than just citing [690b0543a813](<https://git.kernel.org/linus/690b0543a813>), for example, a developer would write: 

```
commit 690b0543a813 ("Documentation: fix formatting to make 's' happy")
```

There are times, Torvalds says, when the hash provided for a commit is incorrect (often because a rebase operation will have caused it to change), but the short changelog can always be used to locate the correct commit in the repository. Tools should support using that extra information; any workflow that relies too heavily on just the commit ID is already broken, he said. Given that even a twelve-digit hash is often ""line noise"", he was unwilling to make it even worse for a questionable gain. 

That response brought an abrupt end to the conversation; the proposed patches will not be merged into the mainline. That ending cut off one other aspect of Uytterhoeven's changes, though. Current kernel documentation is inconsistent about whether hashes should be abbreviated to _exactly_ twelve characters, or to _at least_ that many. That inconsistency is far from the biggest problem in the kernel's documentation, but it still seems worth straightening out at some point.
