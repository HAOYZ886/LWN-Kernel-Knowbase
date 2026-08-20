---
title: Secure deletion and trash bin support
url: https://lwn.net/Articles/212794/
date: "December 5, 2006"
category: "Filesystems-ext4"
author: 
---

A look at the man page for the `chattr` command reveals some interesting functionality; users may set special bits on files to request either that the file be undeletable, or that deletion be "secure" - meaning that the file's contents truly disappear from the disk. The key word here, however, is "request." Those bits have existed for many years, but few - if any - Linux filesystems actually implement those features. The undeletable and secure deletion flags are just placeholders for a "would be nice" feature to be added in the future. Someday. 

That day may be a little closer thanks to [this patch](<http://lwn.net/Articles/212793/>) posted by Nikolai Joukov. It adds support for those two flags to ext4 in a relatively simple and straightforward way. 

The patch works like this: whenever the last link is removed from a file, the undeletable and secure deletion flags are checked. Should either one be set, the file will be moved over to the `.trash/<uid>/` directory in the root of the filesystem. Each per-uid directory has restrictive permissions, keeping users from perusing each others' deleted files. There are no subdirectories, so the path information is lost; preserving paths might be added in a future version. A number is appended to the file name when collisions with files already in the trash happen. 

That's it for the kernel side. Undeletion is easily handled from user space by simply moving the file back out of the trash. The secure deletion feature is also to be done in user space, however. A special daemon can overwrite the file data in whatever way best suits the user's paranoia, then delete the file for real. A possible addition to the patch is a notification mechanism to force that daemon to run when filesystem space gets tight. In any case, all of the policy decisions on how to handle secure deletion requests would live in user space. 

One might wonder why the trash can needs to be implemented in the kernel. The desktop projects have, after all, had a trash can available for some time. There seem to be two reasons why this patch adds that functionality. The first is that it comes for free with this approach to secure deletion. More importantly, however: it is not really possible for a user-space solution to intercept every attempt to delete a file. The nicest file manager available will not be able do do anything about an "`rm`" command typed into a shell, or an `unlink()` call from within a non-cooperating application. Catching file deletion within the kernel ensures that none will slip through the cracks. 

The patch has not received a whole lot of comments as of this writing. One question which has come up is: why not do this at the VFS layer, rather than within ext4? There is little that is ext4-specific about the patch, and doing the work within the VFS would make this feature available to all filesystems - at least those which support the relevant file flags. Mr. Joukov [agrees](<https://lwn.net/Articles/212796/>) that moving this feature up might be the right thing to do, so there may be a reworked version of this patch coming in the future.
