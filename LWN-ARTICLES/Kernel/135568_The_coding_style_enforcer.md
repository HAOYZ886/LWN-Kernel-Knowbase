---
title: The coding style enforcer
url: https://lwn.net/Articles/135568/
date: "May 11, 2005"
category: Coding style
author: 
---

The coding style document packaged with the kernel source contains a number of clear rules; here's one of them: 

Don't put multiple statements on a single line unless you have something to hide: 

```
if (condition) do_this;
              do_something_everytime;
```

Jesper Juhl recently found some code which evidently had something to hide, and [submitted a patch](<https://lwn.net/Articles/135569/>) to break the offending `if` statements onto two lines. Andrew Morton [rejected](<https://lwn.net/Articles/135570/>) it: 

There are about 88 squillion of these in the kernel. I think it would be a mistake for me to start taking such patches, sorry. 

In further discussion, however, Andrew seemed to agree that, perhaps, cleaning up the kernel source to be more generally compliant with the coding style documentation might be a good thing. He just doesn't want to cope with hundreds of little patches to that end. He will, however, consider a small number of very large patches. 

So a major coding style cleanup seems likely to happen, perhaps before 2.6.12 comes out. Applying this sort of patch so late in the cycle _should_ be safe; the intent is to change the formatting, but to make no actual code changes. Andrew also [plans](<https://lwn.net/Articles/135573/>) to drop any changes which do not apply against the -mm tree, in the hopes of minimizing the effects of the changes on patches maintained by other developers. 

If all goes according to this plan, the final 2.6.12 patch could be large indeed.
