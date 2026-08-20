---
title: The AppArmor debate begins
url: https://lwn.net/Articles/181508/
date: "April 26, 2006"
category: "AppArmor; Security-Security technologies"
author: 
---

Novell [announced](<http://lwn.net/Articles/166975/>) the release of the AppArmor security module last January. Then everything went quiet; in particular, no attempt was made to get the AppArmor code merged into the mainline kernel. The silence was broken last week, however, as a result of [the discussion on the possible removal of the Linux security module (LSM) API](<http://lwn.net/Articles/180194/>). The submission of the AppArmor code has had the desired short-term effect: the discussion has moved away from removal of the LSM interface and toward the merits of AppArmor. The AppArmor developers may not see that shift as a blessing at the moment, however. 

As expected, AppArmor has taken a fair amount of criticism. The largest complaint is the fact that AppArmor uses pathnames for its security policies. Using AppArmor, a system administrator can provide a list of files accessible by a given application; anything not on the list becomes inaccessible. Other things - such as capabilities - are also configurable, but there is no controversy over that aspect of the system. It is the use of path names which raises the red flags. 

The sticking point is that a given file name is not the file itself. So, while `/etc/shadow` might identify the shadow password file, that _name_ is not the shadow password file. If an attacker is able to create another name for that file (through the use of links or namespaces, perhaps), that other name could become a way for the attacker to access the shadow password file. So, even if AppArmor forbids access to `/etc/shadow` for a given application, that application might still have access to other pathnames which could be made to refer to the same file. 

AppArmor thus differs from the SELinux approach, which attaches labels to objects and enforces access control rules based on the labels. With SELinux, the shadow password file has the same label (and, thus, the same access rules) regardless of the name by which it is accessed. So SELinux lacks a possible failure mode (rule bypass through different file names) that exists in AppArmor. Of course, as any SELinux administrator knows, maintaining file labels in a consistent and correct state poses challenges of its own. 

The other problem with the AppArmor approach is that the LSM API is not well suited to pathname-based security policies. As a result, AppArmor must often go through a fair amount of (potentially expensive) pain to obtain the names corresponding to files. The impedance mismatch between AppArmor and LSM is not generally seen as a reason to keep AppArmor out of the kernel, but it has led to suggestions that the AppArmor developers should either extend LSM for pathname-based policies or just add their own patches and drop LSM altogether. If AppArmor gets past the other objections, some work will almost certainly have to be done in this area. 

At this point, how any decision will be made on merging AppArmor is far from clear. It has not escaped notice that some of the strongest criticism of AppArmor is coming from the SELinux camp; SELinux developer Stephen Smalley has [defended](<https://lwn.net/Articles/181515/>) that criticism this way: 

We're not threatened by alternatives. We're concerned about a technically unsound approach. The arguments being raised against pathname-based access control are about the soundness of that technical approach, not whether there should be any alternatives to SELinux. 

The proponents of AppArmor claim that the approach is sound. Unlike SELinux, AppArmor does not attempt to be the ultimate security solution for all situations. Instead, it simply puts a lid on applications which might be compromised by an attacker. AppArmor raises the bar by limiting what a broken application might do; it does not attempt to regulate the interactions between every application and every object in the system. This approach is, it is claimed, enough to significantly raise the security of a system while maintaining an administrative interface which is accessible to mere mortals. And, for AppArmor's goals, a pathname-based access control mechanism is said to be good enough. It will probably be some time before we will see whether the kernel development community agrees with that claim. 

(See also: [this detailed criticism of pathname-based access control](<http://securityblog.org/brindle/2006/04/19/security-anti-pattern-path-based-access-control/>) by Joshua Brindle).
