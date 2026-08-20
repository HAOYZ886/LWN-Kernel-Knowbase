---
title: "The abrupt un-exporting of symbols"
url: https://lwn.net/Articles/118835/
date: "January 12, 2005"
category: "Development model-Loadable modules; Modules-Exported symbols; set fs root"
author: 
---

This seems like a conversation we have [seen before](<https://lwn.net/Articles/71710/>): Paul McKenney is asking to have an exported symbol restored for use by an proprietary IBM module. This time around, Paul has submitted [a patch](<https://lwn.net/Articles/118836/>) requesting that two symbols (`files_lock` and `set_fs_root()`) be exported to all modules. It is proving to be a hard sell. 

`files_lock` is a spinlock used within the VFS layer; `set_fs_root()` is used to change the root directory for (one process's view of) a filesystem. They were used by IBM's MVFS to a novel end: MVFS implements a revision control system internally, and allows each process to see a different revision of the file tree. By using these symbols, MVFS was able to make the filesystem behave differently for each process. With 2.6.9, that worked great, but those symbols are no longer exported in 2.6.10. Paul has asked that they be restored so that the MVFS module can work again. 

The export was removed because the kernel developers feel that no code outside of the VFS layer should be making changes in the filesystem namespace. The tricks that MVFS is performing with `set_fs_root()` would be better done with bind mounts - in user space. It is also felt that any code using `set_fs_root()` or `files_lock` can only be a fundamental part of the kernel, and thus a derived product; there is no legal way, according to the relevant kernel developers, that a proprietary module can legally use them. For these reasons, the exports were removed, and there is strong resistance to restoring them. 

Nobody disagrees with the reasoning behind the change. Not everybody thinks that it was appropriate to remove the symbols with no notice, however. In particular, Linus [thinks there was no reason](<https://lwn.net/Articles/118838/>) to break things so abruptly: 

I'm known for happily breaking binary modules, but I think we should do it only if we have a reason _other_ than "let's break a module". 

Andrew Morton also [thinks the exports should be restored](<https://lwn.net/Articles/118840/>) for a period of time - a position which gained him [an accusation](<https://lwn.net/Articles/118841/>) of supporting IBM's position as a payback for IBM's funding of OSDL. Despite Linus's and Andrew's position, as of this writing, the exports of those symbols have not been restored. 

This whole episode restarted the discussion of what the proper way is to remove deprecated features when there is no unstable kernel series in sight. Andrew proposed the creation of a file (`feature-removal-schedule.txt`) in the `Documentation` directory which would list things slated for removal, and the relevant dates. That file [has been created](<https://lwn.net/Articles/118844/>); as of this writing it lists devfs and some CPU frequency files in `/proc`. This file will be helpful for some users, but it probably will not make life easier for people maintaining out-of-tree code; Christoph Hellwig and others have made it clear that they will continue to remove "unneeded" exports without notice as they are identified. Life will continue to be difficult, it seems, for code maintained outside of the mainline tree.
