---
title: "RHEL, kernel vulnerabilities, and days of risk"
url: https://lwn.net/Articles/128853/
date: "March 23, 2005"
category: Security
author: 
---

Security Innovation has joined the elite group of Microsoft-funded researchers who somehow manage to reach pro-Microsoft conclusions. This company's latest output is [a report on the relative security of Linux and Windows web servers [PDF]](<http://www.sisecure.com/pdf/windows_linux_final_study.pdf>) which states that Windows is more secure, in this role, than Red Hat Enterprise Linux. The group did its work by looking at all of the vulnerabilities fixed by each vendor in 2004 (as designated by CVE numbers), and determining how much time passed between the initial disclosure of the problem and the resulting fix. Windows showed fewer vulnerabilities, and significantly fewer "days of risk" when disclosed problems lacked a patch. 

Those who want to poke holes in this study should be able to find ample opportunity. Microsoft vulnerabilities are less likely to be disclosed prior to patching, to the point that the median "days of risk" for Windows was zero. The report cautions against writing off "low risk" vulnerabilities, but, somehow, Microsoft simply does not _have_ any "low risk" problems. Either that, or Microsoft doesn't bother to fix them, resulting in many undisclosed "days of risk." Red Hat will also have gotten burned by [this libpng vulnerability](<http://lwn.net/Articles/90942/>), which, by mistake, remained unfixed for two years. That's a lot of days of risk, even though no known exploits of this vulnerability took place. 

Let's focus on one specific claim, however: 

There were thirty one [RHEL] vulnerabilities fixed in 2004 that had more than 90 days of risk, and of these, seven were designated by ICAT as high severity... Eleven of these vulnerabilities were in the operating system kernel. 

The report does not list the actual vulnerabilities it looked at, so we'll have to try to reproduce that work ourselves. Here's the kernel vulnerabilities fixed by Red Hat in 2004: 

CAN # | Disclosed | Fixed | Days | Description  
---|---|---|---|---  
[CVE-2004-0001](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2004-0001>) | 2004-1-16 | [2004-1-16](<https://rhn.redhat.com/errata/RHSA-2004-017.html>) | 0 | x86-64 ptrace bug  
[CVE-2004-0077](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2004-0077>) | 2004-2-18 | [2004-2-20](<https://rhn.redhat.com/errata/RHSA-2004-066.html>) | 2 | `mremap()` local root exploit  
[CAN-2004-0109](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0109>) | 2004-4-14 | [2004-4-22](<https://rhn.redhat.com/errata/RHSA-2004-183.html>) | 8 | ISO9660 buffer overflow  
[CAN-2004-0424](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0424>) | 2004-4-20 | [2004-4-22](<https://rhn.redhat.com/errata/RHSA-2004-183.html>) | 2 | `ip_setsockopt()` local root exploit  
[CAN-2003-0461](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2003-0461>) | 2002-5-2 | [2004-5-11](<https://rhn.redhat.com/errata/RHSA-2004-188.html>) | 737 | TTY char count information leak  
[CAN-2003-0465](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2003-0465>) | 2003-7-11 | [2004-5-11](<https://rhn.redhat.com/errata/RHSA-2004-188.html>) | 305 | `strncpy()` potential information leak  
[CAN-2003-0984](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2003-0984>) | 2003-12-4 | [2004-5-11](<https://rhn.redhat.com/errata/RHSA-2004-188.html>) | 159 | RTC information leak  
[CAN-2003-1040](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2003-1040>) | 2003-12-4 | [2004-5-11](<https://rhn.redhat.com/errata/RHSA-2004-188.html>) | 159 | kmod local denial of service  
[CAN-2004-0003](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0003>) | 2004-1-15 | [2004-5-11](<https://rhn.redhat.com/errata/RHSA-2004-188.html>) | 116 | DRI range checking  
[CAN-2004-0010](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0010>) | 2004-2-18 | [2004-5-11](<https://rhn.redhat.com/errata/RHSA-2004-188.html>) | 83 | ncpfs buffer overflow  
[CAN-2004-0427](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0427>) | 2004-4-8 | [2004-6-17](<https://rhn.redhat.com/errata/RHSA-2004-255.html>) | 70 |   
[CAN-2004-0495](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0495>) | 2004-6-17 | [2004-6-17](<https://rhn.redhat.com/errata/RHSA-2004-255.html>) | 0 | Potential driver bugs found by sparse  
[CAN-2004-0554](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0554>) | 2004-6-9 | [2004-6-17](<https://rhn.redhat.com/errata/RHSA-2004-255.html>) | 8 | Floating point denial of service  
[CAN-2004-0497](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0497>) | 2004-7-2 | [2004-7-2](<https://rhn.redhat.com/errata/RHSA-2004-354.html>) | 0 | NFS group permissions  
[CAN-2004-0178](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0178>) | 2004-3-8 | [2004-8-3](<https://rhn.redhat.com/errata/RHSA-2004-413.html>) | 148 | SoundBlaster denial of service  
[CAN-2004-0415](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0415>) | 2004-8-3 | [2004-8-3](<https://rhn.redhat.com/errata/RHSA-2004-413.html>) | 0 | 64-bit information leak  
[CAN-2004-0447](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0447>) | 2004-6-19 | [2004-8-3](<https://rhn.redhat.com/errata/RHSA-2004-413.html>) | 45 | ia-64 denial of service  
[CAN-2004-0535](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0535>) | 2004-6-3 | [2004-8-3](<https://rhn.redhat.com/errata/RHSA-2004-413.html>) | 61 | e1000 driver information leak  
[CAN-2004-0587](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0587>) | 2004-5-4 | [2004-8-3](<https://rhn.redhat.com/errata/RHSA-2004-413.html>) | 91 | qla driver denial of service  
[CAN-2004-0136](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0136>) | 2004-6-14 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 171 | ELF binary denial of service  
[CAN-2004-0619](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0619>) | 2004-6-23 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 162 | Broadcom 5820 driver buffer overflow  
[CAN-2004-0685](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0685>) | 2004-8-25 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 99 | USB driver information leak  
[CAN-2004-0812](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0812>) | 2004-11-8 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 24 | x86_64 TSS error  
[CAN-2004-0883](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0883>) | 2004-11-17 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 15 | smbfs remotely exploitable vulnerabilities  
[CAN-2004-0949](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0949>) | 2004-11-17 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 15 | smbfs packet reassembly  
[CAN-2004-1068](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1068>) | 2004-11-19 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 13 | Datagram serializing problem  
[CAN-2004-1070](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1070>) | 2004-11-10 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 22 | ELF loader overflow  
[CAN-2004-1071](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1071>) | 2004-11-10 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 22 | ELF loader `mmap()` failure  
[CAN-2004-1072](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1072>) | 2004-11-10 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 22 | ELF loader interpreter name buffer overflow  
[CAN-2004-1073](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1073>) | 2004-11-10 | [2004-12-2](<https://rhn.redhat.com/errata/RHSA-2004-549.html>) | 22 | ELF loader file disclosure  
[CAN-2004-0565](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-0565>) | 2004-5-28 | [2004-12-23](<https://rhn.redhat.com/errata/RHSA-2004-689.html>) | 209 | ia-64 floating point information leak  
[CAN-2004-1016](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1016>) | 2004-12-14 | [2004-12-23](<https://rhn.redhat.com/errata/RHSA-2004-689.html>) | 9 | `sendmsg()` denial of service  
[CAN-2004-1017](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1017>) | 2004-12-10 | [2004-12-23](<https://rhn.redhat.com/errata/RHSA-2004-689.html>) | 13 | Edgeport driver buffer overflow  
[CAN-2004-1137](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1137>) | 2004-12-14 | [2004-12-23](<https://rhn.redhat.com/errata/RHSA-2004-689.html>) | 9 | IGMP remote exploit  
[CAN-2004-1144](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1144>) | 2004-12-22 | [2004-12-23](<https://rhn.redhat.com/errata/RHSA-2004-689.html>) | 1 | x86_64 32-bit emulation local root exploit  
[CAN-2004-1234](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1234>) | 2004-4-8 | [2004-12-23](<https://rhn.redhat.com/errata/RHSA-2004-689.html>) | 113 | ELF denial of service  
[CAN-2004-1335](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2004-1335>) | 2004-12-15 | [2004-12-23](<https://rhn.redhat.com/errata/RHSA-2004-689.html>) | 8 | IP options integer overflow  
  
The attentive reader may have noticed that this is a rather long list of vulnerabilities. Summed up, it amounts to a total of 2943 days of risk - a substantial portion of the 12,415 days of risk cited in the report. 

One immediate conclusion is that, in many cases, we are talking about "days of very low risk." The `strncpy()` information leak was worth fixing, but few people were likely to be overly worried during the 305 days it took for Red Hat to issue updates with that fix. The same is true of the TTY character count leak (737 days of risk). Both ia-64 users could probably live with the floating point leak on that platform (209 days of risk). In other words, many of the vulnerabilities which had a big contribution to the total number of days of risk were of little concern. 

On the other hand, Red Hat was slow in fixing some important problems. The kmod denial of service and ELF vulnerabilities took months to fix - and they were clearly (locally) exploitable problems. Red Hat is, at times, leaving its paying customers with known security problems for longer than it should. 

Interestingly, many of these problems were fixed more quickly in other distributions - including Fedora Core. Red Hat's stability goals for its Enterprise Linux line could be an issue here. The need for more stress and regression testing of kernel updates, combined with a clear wish to minimize the number of disruptive kernel updates (many updates fixed several vulnerabilities), is causing those updates to be delayed. Thus, one might draw the ironic conclusion that, if you want the fastest security updates, you're better off not paying for them. 

There are some more predictable conclusions as well. One is that reports like the one from Security Innovation still do not mean a whole lot. There are too many variables; it is hard to get a handle on which system is truly more secure, and it is too easy to tilt the data in one direction or the other. Of course, one _could_ look at the number and cost of actual security incidents, but these Microsoft-funded surveys tend not to do that. The final, predictable conclusion is this: regardless of how Linux performs relative to other systems, we are not doing nearly well enough. As long as we are producing such long lists of bugs (for a single system component), our claims to security will only hold so much water.
