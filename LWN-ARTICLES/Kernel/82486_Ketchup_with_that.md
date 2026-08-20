---
title: "Ketchup with that?"
url: https://lwn.net/Articles/82486/
date: "April 28, 2004"
category: Development tools; Ketchup
author: 
---

Matt Mackall has [released](<https://lwn.net/Articles/82044/>) version 0.7 of his "ketchup" script. Ketchup can be thought of as a sort of apt-get for kernel trees; run "`ketchup 2.6-bk`" and it will go get the right combination of kernel tarballs and patch sets and put them together into a complete kernel tree. Several different trees are supported, including `-mm`, `-tiny`, and `-mjb`, and the script can string together a series of patches to get to the desired destination. If you find yourself playing with a number of different kernel trees, ketchup may prove to be a tasty condiment to add to your tool collection.
