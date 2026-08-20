---
title: "A weak cryptoloop implementation in Linux?"
url: https://lwn.net/Articles/67216/
date: "January 21, 2004"
category: "Block layer-Loopback device; Cryptoloop; Security"
author: 
---

The "cryptoloop" code in the Linux kernel allows "loopback" mounts of filesystems. Essentially, cryptoloop looks like a block driver which encrypts data on its way through. It can thus be used to add encryption to any of the standard Linux filesystems without changing the filesystem code itself. 

Recently, in response to a bug report with the 2.6.1-mm3 cryptoloop implementation, Jari Ruusu made [a disturbing claim](<https://lwn.net/Articles/67217/>): 

If you want your data secure, you need to re-encrypt your data anyway. Mainline loop crypto implementation has exploitable vulnerability that is equivalent to back door. Kerneli.org folks have always shipped back-doored loop crypto, and now mainline folks are shipping back-doored loop crypto. Kerneli.org derivatives such as Debian, SuSE, and others are also back-doored. 

It will come as no surprise that this message was followed by requests for more details on the "back-doored" cryptoloop. Jari [obliged](<https://lwn.net/Articles/67223/>) with a clear, technical explanation of what is going on. If you are using (or considering) cryptoloop. it is worth a look, even if there may be no need for immediate panic. 

The problem, it seems, is that cryptoloop is susceptible to a certain kind of known plaintext attack. For any given filesystem type, the contents of certain sectors will be easy to predict. Given some time and an idle processor, an attacker can generate an exhaustive dictionary of likely passwords and the resulting ciphertext that will appear on disk. With access to the actual, encrypted disk, a quick lookup in the dictionary will yield the password and enable decryption of the entire filesystem. This attack is not practical for casual snoopers, but it would not be entirely surprising if government agencies and other, relatively organized groups had this sort of dictionary handy. 

There are two ways of getting around this sort of problem. One is to choose a lengthy, non-obvious password. The other is to use salted passwords, where the password is modified by a randomly-chosen value before the data is encrypted. The salt value has to be retrievable, but it has the effect of requiring an attacker to create a separate dictionary for every possible number. If the range of salt values is large enough, salting the password will render the dictionary attack impractical. 

The end result is that most cryptoloop users need not go into an immediate panic, but this weakness is worth being aware of. It would also be a good idea to get a stronger mechanism into the mainline kernel. There is little to be gained and much to be lost by shipping crypto code with known weaknesses.
