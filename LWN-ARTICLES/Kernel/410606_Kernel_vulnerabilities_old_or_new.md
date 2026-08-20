---
title: "Kernel vulnerabilities: old or new?"
url: https://lwn.net/Articles/410606/
date: "October 19, 2010"
category: "Security-Vulnerabilities"
author: "By Jonathan Corbet October 19, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 19, 2010

A quick [search of the CVE database](<http://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=linux+kernel+2010>) turns up 80 CVE numbers related to kernel vulnerabilities so far this year. At one recent conference or another, while talking with a prominent kernel developer, your editor confessed that he found that number to be discouragingly high. In an era where there is clearly an increasing level of commercial, criminal, and governmental interest in exploiting security holes, it would be hard to be doing enough to avoid the creation of vulnerabilities. But, your editor wondered, could we be doing more than we are? The response your editor got was, in essence, that the bulk of the holes being disclosed were ancient vulnerabilities which were being discovered by new static analysis tools. In other words, we are fixing security problems faster than we are creating them. 

That sort of claim requires verification; it is also amenable to being verified by a researcher with sufficient determination and pain resistance. Your editor decided to give it a try. "All" that would be required, after all, was to look at each vulnerability and figure out when it was introduced. How hard could that be? 

So, the basic process followed was this: pick a CVE entry, find the patch which closed the hole, then dig through the repository history and other resources in an attempt to figure out just when the problem was first introduced into the kernel. In some cases, the answer was relatively easy to find; others were sufficiently hard that your editor eventually gave up. One especially valuable resource in the search turned out to be the Red Hat bugzilla; the developers there (and Eugene Teo in particular) go out of their way to document the particulars of vulnerabilities. Sometimes, the commit which introduced the bug was simply listed there. The "git gui blame" utility is also quite useful when doing this kind of research. 

About 60 of the 80 vulnerabilities listed above were dealt with in this way before your editor's eyes crossed permanently. The results can be seen in the following table. Let it be said from the outset that there will inevitably be some errors in the data below; the most likely mistake will be assigning blame to a commit which actually just moved the vulnerability from somewhere else. That may lead to a bias that makes vulnerabilities look more recent than they really are. That said, a best effort has been made, and things should not be too far off. 

> CVE # | Introduced | Fixed  
> ---|---|---  
> Commit| Release| Commit| Release  
> [CVE-2010-3477](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3477>) | \-- | <2.6.13 | [0f04cfd0](<http://git.kernel.org/linus/0f04cfd098fb81fded74e78ea1a1b86cc6c6c31e>) | 2.6.36  
> [CVE-2010-3442](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3442>) | \-- | <2.6.13 | [5591bf07](<http://git.kernel.org/linus/5591bf07225523600450edd9e6ad258bb877b779>) | 2.6.36  
> [CVE-2010-3437](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3437>) | \-- | <2.6.13 | [252a52aa](<http://git.kernel.org/linus/252a52aa4fa22a668f019e55b3aac3ff71ec1c29>) | 2.6.36  
> [CVE-2010-3310](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3310>) | \-- | <2.6.13 | [9828e6e6](<http://git.kernel.org/linus/9828e6e6e3f19efcb476c567b9999891d051f52f>) | 2.6.36  
> [CVE-2010-3301](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3301>) | [d4d67150](<http://git.kernel.org/linus/d4d67150165df8bf1cc05e532f6efca96f907cab>) | 2.6.27 | [36d001c7](<http://git.kernel.org/linus/36d001c70d8a0144ac1d038f6876c484849a74de>) | 2.6.36  
>    
> [CVE-2010-3298](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3298>) | [542f5482](<http://git.kernel.org/linus/542f54823614915780c3459b0e6062f06c0c0f99>) | 2.6.29 | [7011e660](<http://git.kernel.org/linus/7011e660938fc44ed86319c18a5954e95a82ab3e>) | 2.6.36  
> [CVE-2010-3297](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3297>) | \-- | <2.6.13 | [44467187](<http://git.kernel.org/linus/44467187dc22fdd33a1a06ea0ba86ce20be3fe3c>) | 2.6.36  
> [CVE-2010-3296](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3296>) | [4d22de3e](<http://git.kernel.org/linus/4d22de3e6cc4a09c369b504cd8bcde3385a974cd>) | 2.6.21 | [49c37c03](<http://git.kernel.org/linus/49c37c0334a9b85d30ab3d6b5d1acb05ef2ef6de>) | 2.6.36  
> [CVE-2010-3084](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3084>) | [2d96cf8c](<http://git.kernel.org/linus/2d96cf8cdfd625da51e5d237d564cd75d3147547>) | 2.6.30 | [ee9c5cfa](<http://git.kernel.org/linus/ee9c5cfad29c8a13199962614b9b16f1c4137ac9>) | 2.6.36  
> [CVE-2010-3081](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3081>) | [42908c69](<http://git.kernel.org/linus/42908c69f61f75dd70e424263ab89ee52040382b>) | 2.6.26 | [c41d68a5](<http://git.kernel.org/linus/c41d68a513c71e35a14f66d71782d27a79a81ea6>) | 2.6.36  
>    
> [CVE-2010-3080](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3080>) | [7034632d](<http://git.kernel.org/linus/7034632d88b02960abee258056d2269e606707e9>) | 2.6.24 | [27f7ad53](<http://git.kernel.org/linus/27f7ad53829f79e799a253285318bff79ece15bd>) | 2.6.36  
> [CVE-2010-3079](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3079>) | [5072c59f](<http://git.kernel.org/linus/5072c59fd45e9976d02ee6f18c7336ef97623cbc>) | 2.6.27 | [9c55cb12](<http://git.kernel.org/linus/9c55cb12c1c172e2d51e85fbb5a4796ca86b77e7>) | 2.6.36  
> [CVE-2010-3078](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3078>) | \-- | <2.6.13 | [a122eb2f](<http://git.kernel.org/linus/a122eb2fdfd78b58c6dd992d6f4b1aaef667eef9>) | 2.6.36  
> [CVE-2010-3067](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3067>) | \-- | <2.6.13 | [75e1c70f](<http://git.kernel.org/linus/75e1c70fc31490ef8a373ea2a4bea2524099b478>) | 2.6.36  
> [CVE-2010-3015](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3015>) | unknown | [731eb1a0](<http://git.kernel.org/linus/731eb1a03a8445cde2cb23ecfb3580c6fa7bb690>) | 2.6.34  
>    
> [CVE-2010-2960](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2960>) | [ee18d64c](<http://git.kernel.org/linus/ee18d64c1f632043a02e6f5ba5e045bb26a5465f>) | 2.6.32 | [3d96406c](<http://git.kernel.org/linus/3d96406c7da1ed5811ea52a3b0905f4f0e295376>) | 2.6.36  
> [CVE-2010-2959](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2959>) | [ffd980f9](<http://git.kernel.org/linus/ffd980f976e7fd666c2e61bf8ab35107efd11828>) | 2.6.25 | [5b75c497](<http://git.kernel.org/linus/5b75c4973ce779520b9d1e392483207d6f842cde>) | 2.6.36  
> [CVE-2010-2955](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2955>) | [3d23e349](<http://git.kernel.org/linus/3d23e349d807177eaf519d444677cee86b1a04cf>) | 2.6.33 | [42da2f94](<http://git.kernel.org/linus/42da2f948d949efd0111309f5827bf0298bcc9a4>) | 2.6.36  
> [CVE-2010-2946](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2946>) | \-- | <2.6.13 | [aca0fa34](<http://git.kernel.org/linus/aca0fa34bdaba39bfddddba8ca70dba4782e8fe6>) | 2.6.36  
> [CVE-2010-2943](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2943>) | \-- | <2.6.13 | [7124fe0a](<http://git.kernel.org/linus/7124fe0a5b619d65b739477b3b55a20bf805b06d>) | 2.6.35  
>    
> [CVE-2010-2942](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2942>) | \-- | 2.6.9 | [1c40be12](<http://git.kernel.org/linus/1c40be12f7d8ca1d387510d39787b12e512a7ce8>) | 2.6.36  
> [CVE-2010-2803](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2803>) | unknown | [b9f0aee8](<http://git.kernel.org/linus/b9f0aee83335db1f3915f4e42a5e21b351740afd>) | 2.6.36  
> [CVE-2010-2798](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2798>) | [71b86f56](<http://git.kernel.org/linus/71b86f562b5eb6f94ea00bba060caa64d0137969>) | 2.6.19 | [728a756b](<http://git.kernel.org/linus/728a756b8fcd22d80e2dbba8117a8a3aafd3f203>) | 2.6.35  
> [CVE-2010-2653](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2653>) | \-- | <2.6.13 | [e74d098c](<http://git.kernel.org/linus/e74d098c66543d0731de62eb747ccd5b636a6f4c>) | 2.6.34  
> [CVE-2010-2538](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2538>) | [e441d54d](<http://git.kernel.org/linus/e441d54de4fd97dd381f3e73636f5ba51ff4c7d9>) | 2.6.29 | [2ebc3464](<http://git.kernel.org/linus/2ebc3464781ad24474abcbd2274e6254689853b5>) | 2.6.35  
>    
> [CVE-2010-2537](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2537>) | [c5c9cd4d](<http://git.kernel.org/linus/c5c9cd4d1b827fe545ed2a945e91e3a6909f3886>) | 2.6.29 | [2ebc3464](<http://git.kernel.org/linus/2ebc3464781ad24474abcbd2274e6254689853b5>) | 2.6.35  
> [CVE-2010-2524](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2524>) | [6103335d](<http://git.kernel.org/linus/6103335de8afa5d780dcd512abe85c696af7b040>) | 2.6.25 | [4c0c03ca](<http://git.kernel.org/linus/4c0c03ca54f72fdd5912516ad0a23ec5cf01bda7>) | 2.6.35  
> [CVE-2010-2521](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2521>) | \-- | <2.6.13 | [2bc3c117](<http://git.kernel.org/linus/2bc3c1179c781b359d4f2f3439cb3df72afc17fc>) | 2.6.34  
> [CVE-2010-2492](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2492>) | [dd2a3b7a](<http://git.kernel.org/linus/dd2a3b7ad98f8482cae481cad89dfed5eee48365>) | 2.6.21 | [a6f80fb7](<http://git.kernel.org/linus/a6f80fb7b5986fda663d94079d3bba0937a6b6ff>) | 2.6.35  
> [CVE-2010-2478](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2478>) | [0853ad66](<http://git.kernel.org/linus/0853ad66b14feb12acde7ac13b7c3b75770a0adc>) | 2.6.27 | [db048b69](<http://git.kernel.org/linus/db048b69037e7fa6a7d9e95a1271a50dc08ae233>) | 2.6.35  
>    
> [CVE-2010-2248](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2248>) | \-- | <2.6.13 | [6513a81e](<http://git.kernel.org/linus/6513a81e9325d712f1bfb9a1d7b750134e49ff18>) | 2.6.34  
> [CVE-2010-2240](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2240>) | \-- | <2.6.13 | [320b2b8d](<http://git.kernel.org/linus/320b2b8de12698082609ebbc1a17165727f4c893>) | 2.6.35  
> [CVE-2010-2226](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2226>) | [f6aa7f21](<http://git.kernel.org/linus/f6aa7f2184330262e1cb5f7802536e5346bd46a3>) | 2.6.25 | [1817176a](<http://git.kernel.org/linus/1817176a86352f65210139d4c794ad2d19fc6b63>) | 2.6.35  
> [CVE-2010-2071](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2071>) | [744f52f9](<http://git.kernel.org/linus/744f52f997b21de2a01931f79cf6c198bf688120>) | 2.6.29 | [2f26afba](<http://git.kernel.org/linus/2f26afba46f0ebf155cf9be746496a0304a5b7cf>) | 2.6.35  
> [CVE-2010-2066](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2066>) | [748de673](<http://git.kernel.org/linus/748de6736c1e482e111f9d1b5a5d5b1787600cad>) | 2.6.31 | [1f5a81e4](<http://git.kernel.org/linus/1f5a81e41f8b1a782c68d3843e9ec1bfaadf7d72>) | 2.6.35  
>    
> [CVE-2010-1643](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1643>) | \-- | <2.6.13 | [731572d3](<http://git.kernel.org/linus/731572d39fcd3498702eda4600db4c43d51e0b26>) | 2.6.28  
> [CVE-2010-1641](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1641>) | [71b86f56](<http://git.kernel.org/linus/71b86f562b5eb6f94ea00bba060caa64d0137969>) | 2.6.19 | [7df0e039](<http://git.kernel.org/linus/7df0e0397b9a18358573274db9fdab991941062f>) | 2.6.35  
> [CVE-2010-1636](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1636>) | [f2eb0a24](<http://git.kernel.org/linus/f2eb0a241f0e5c135d93243b0236cb1f14c305e0>) | 2.6.29 | [5dc64164](<http://git.kernel.org/linus/5dc6416414fb3ec6e2825fd4d20c8bf1d7fe0395>) | 2.6.34  
> [CVE-2010-1488](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1488>) | [28b83c51](<http://git.kernel.org/linus/28b83c5193e7ab951e402252278f2cc79dc4d298>) | 2.6.32 | [b95c35e7](<http://git.kernel.org/linus/b95c35e76b29ba812e5dabdd91592e25ec640e93>) | 2.6.34  
> [CVE-2010-1437](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1437>) | \-- | <2.6.13 | [cea7daa3](<http://git.kernel.org/linus/cea7daa3589d6b550546a8c8963599f7c1a3ae5c>) | 2.6.34  
>    
> [CVE-2010-1436](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1436>) | [18ec7d5c](<http://git.kernel.org/linus/18ec7d5c3f434aed9661ed10a9e1f48cdeb4981d>) | 2.6.19 | [7e619bc3](<http://git.kernel.org/linus/7e619bc3e6252dc746f64ac3b486e784822e9533>) | 2.6.35  
> [CVE-2010-1188](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1188>) | \-- | <2.6.13 | [fb7e2399](<http://git.kernel.org/linus/fb7e2399ec17f1004c0e0ccfd17439f8759ede01>) | 2.6.20  
> [CVE-2010-1173](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1173>) | \-- | <2.6.13 | [5fa782c2](<http://git.kernel.org/linus/5fa782c2f5ef6c2e4f04d3e228412c9b4a4c8809>) | 2.6.34  
> [CVE-2010-1162](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1162>) | \-- | <2.6.13 | [6da8d866](<http://git.kernel.org/linus/6da8d866d0d39e9509ff826660f6a86a6757c966>) | 2.6.34  
> [CVE-2010-1148](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1148>) | [c3b2a0c6](<http://git.kernel.org/linus/c3b2a0c640bff7df85d79fb4f89674949a267ec2>) | 2.6.29 | [fa588e0c](<http://git.kernel.org/linus/fa588e0c57048b3d4bfcd772d80dc0615f83fd35>) | 2.6.35  
>    
> [CVE-2010-1146](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1146>) | [73422811](<http://git.kernel.org/linus/73422811d290c628b4ddbf6830e5cd6fa42e84f1>) | 2.6.31 | [cac36f70](<http://git.kernel.org/linus/cac36f707119b792b2396aed371d6b5cdc194890>) | 2.6.34  
> [CVE-2010-1087](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1087>) | \-- | <2.6.13 | [9f557cd8](<http://git.kernel.org/linus/9f557cd8073104b39528794d44e129331ded649f>) | 2.6.33  
> [CVE-2010-1086](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1086>) | \-- | <2.6.13 | [29e1fa35](<http://git.kernel.org/linus/29e1fa3565a7951cc415c634eb2b78dbdbee151d>) | 2.6.34  
> [CVE-2010-1085](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1085>) | [9ad593f6](<http://git.kernel.org/linus/9ad593f6d326e7a4664e3856520f6c042f82a37f>) | 2.6.27 | [fed08d03](<http://git.kernel.org/linus/fed08d036f2aabd8d0c684439de37f8ebec2bbc2>) | 2.6.33  
> [CVE-2010-1084](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1084>) | [be9d1227](<http://git.kernel.org/linus/be9d122730c878baafe11e70d1436faac229f2fc>) | 2.6.15 | [101545f6](<http://git.kernel.org/linus/101545f6fef4a0a3ea8daf0b5b880df2c6a92a69>) | 2.6.34  
>    
> [CVE-2010-1083](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1083>) | \-- | <2.6.13 | [d4a4683c](<http://git.kernel.org/linus/d4a4683ca054ed9917dfc9e3ff0f7ecf74ad90d6>) | 2.6.33  
> [CVE-2010-0622](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-0622>) | [c87e2837](<http://git.kernel.org/linus/c87e2837be82df479a6bae9f155c43516d2feebc>) | 2.6.18 | [51246bfd](<http://git.kernel.org/linus/51246bfd189064079c54421507236fd2723b18f3>) | 2.6.33  
> [CVE-2010-0415](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-0415>) | [742755a1](<http://git.kernel.org/linus/742755a1d8ce2b548428f7aacf1758b4bba50080>) | 2.6.18 | [6f5a55f1](<http://git.kernel.org/linus/6f5a55f1a6c5abee15a0e878e5c74d9f1569b8b0>) | 2.6.33  
> [CVE-2010-0410](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-0410>) | [7672d0b5](<http://git.kernel.org/linus/7672d0b54411371e0b6a831c1cb2f0ce615de6dc>) | 2.6.14 | [f98bfbd7](<http://git.kernel.org/linus/f98bfbd78c37c5946cc53089da32a5f741efdeb7>) | 2.6.33  
> [CVE-2010-0307](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-0307>) | unknown | [221af7f8](<http://git.kernel.org/linus/221af7f87b97431e3ee21ce4b0e77d5411cf1549>) | 2.6.33  
  
Some other notes relevant to the table: 

  * No attempt was made to find the origin of vulnerabilities which were present in the initial commit which began the git era during the 2.6.12 development cycle. Anything which was already present then can certainly be said to be an old bug. 

  * Some parts of the code have been changed so many times that it can be truly hard to determine when a vulnerability was introduced; places where your editor give up are marked as "unknown" above. One could maybe come up with a real answer by bisecting and trying exploits, but your editor's dedication to the task was not quite that strong. 

  * A couple of these bugs are old in a different way - [CVE-2010-1188](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-1188>) was fixed in 2008, but was only understood to be a security issue in 2010\. Anybody running a current kernel would not be vulnerable, but bugs like this can be nicely preserved in enterprise kernels for many years. 

Looking at when the vulnerabilities were introduced yields a chart like this: 

> ![\[Kernel vulnerabilities\]](https://static.lwn.net/images/2010/kernel-vulnerabilities.png)

So, in a sense, the above-mentioned kernel hacker was correct - an awful lot of the vulnerabilities fixed over the last year predate the git era, and are thus over five years old. It seems that security bugs can lurk in the kernel for a very long time before somebody stumbles across them - or, at least, before somebody reports them. 

According to the information above, we have fixed dozens of vulnerabilities since 2.6.33 without introducing any. The latter part of that claim might be charitably described as being unlikely to stand the test of time. There were (at least) 13 vulnerabilities fixed in the 2.6.35 cycle, 21 in the 2.6.36 cycle. We can hope that fewer vulnerabilities were added in that time; it seems certain, though, that (1) the number of vulnerabilities added will not be zero, and (2) it will probably take us five years or more to find many of them. 

There may be some comfort in knowing that a large proportion of 2010's known security vulnerabilities are not a product of 2010's development. Indeed, assuming that a fair number of the old vulnerabilities are a bit older yet, one can also claim that they are not a product of the "new" kernel development model adopted in the early 2.6 days. That claim could be tested by extending this research back into the BitKeeper era; that is a task for a future project. 

Your editor remains concerned, though, that it is too easy to put insecure code into the kernel and too hard to discover the vulnerabilities that are created. Analysis tools can help, but there really is no substitute for painstaking and meticulous code review when it comes to keeping vulnerabilities out of the kernel. At times, it is clear that the amount of review being done is not what it should be. There may well come a day when we'll wish we had found a way to be a bit more careful.
