---
title: Outreach program for women—kernel edition
url: https://lwn.net/Articles/553693/
date: "June 12, 2013"
category: "Development model-Diversity"
author: "By Jake Edge June 12, 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
June 12, 2013

While three kernel internships for women were [originally announced](<https://www.linux.com/news/featured-blogs/167-amanda-mcpherson/716409-announcing-outreach-program-for-women-internships-for-the-linux-kernel-please-apply/>) in late April, the size of the program has more than doubled since then. Seven internships have been established for kernel work through the [Outreach Program for Women](<https://live.gnome.org/OutreachProgramForWomen>) (OPW); each comes with a $5000 stipend and a $500 travel grant. The program officially kicks off on June 17, but the application process already brought in several hundred patch submissions from eighteen applicants, 137 of which were accepted into the staging and Xen trees—all in thirteen days. 

The program was initiated by the Linux Foundation, which found sponsors for the first three slots, but Intel's Open Source Technology Center added three more while the OPW itself came up with funding for another. The OPW has expanded well beyond its GNOME project roots, with eighteen different organizations (e.g. Debian, KDE, Mozilla, Perl, Twisted, and many more) participating in this round. 

The program pairs the interns with a mentor from a participating project to assist the intern with whatever planned work she has taken on for the three months of the each program round. OPW is patterned after the Google Summer of Code project, but is not only for students and programmers as other kinds of projects (and applicants) are explicitly allowed. As the name would imply, it also restricts applicants to those who self-identify as a woman. 

The kernel effort has been guided by Sarah Sharp, who is a USB 3.0 kernel hacker for Intel. She is also one of the mentors for this round. In late May, she put together a [blog post](<http://sarah.thesharps.us/2013/05/23/%EF%BB%BF%EF%BB%BFopw-update/>) that described the application process and the patches it brought in. Sharp filled us in on the chosen interns. In addition, most of the patches accepted can be seen in her cherry-picked [kernel git tree](<https://git.kernel.org/cgit/linux/kernel/git/sarah/xhci.git/log/?h=opw-kernel>). 

#### The interns

Sharp will be mentoring Ksenia (Xenia) Ragiadakou who will be working on the USB 3.0 host driver. Ragiadakou is currently studying for her bachelor's degree in computer science at the University of Crete in Greece. In addition to her cleanup patches for the rtl8192u wireless staging driver, Ragiadakou has already [found a bug](<http://marc.info/?l=linux-usb&m=137052855612729&w=2>) in Sharp's host controller driver. 

Two of the interns will be working on the Xen subsystem of the kernel with mentors Konrad Wilk of Oracle and Stefano Stabellini of Citrix. They are Lisa T. Nguyen, who received a bachelor's degree in computer science from the University of Washington in 2007, and Elena Ufimtseva, who got a master's degree in computer science from St. Petersburg University of Information Technologies in 2006. Nguyen did several [cleanup patches](<https://groups.google.com/forum/#!searchin/opw-kernel/xen>) for Xen (along with various other cleanups) as part of the application process, while Ufimtseva focused on cleanups in the [ced1401](<https://groups.google.com/forum/#!searchin/opw-kernel/ced1401>) (Cambridge Electronics 1401 USB device) driver in staging. 

Lidza Louina will be working with Greg Kroah-Hartman as a mentor on further cleanups in staging drivers. She was working on a bachelor's degree in computer science at the University of Massachusetts but had to take time off to work full-time. Her contributions were to the csr wireless driver in the staging tree. 

Tülin İzer is working on parallelizing the x86 boot process with mentor PJ Waskiewicz of Intel. She is currently pursuing a bachelor's degree in computer engineering at Galatasary University in Istanbul, Turkey. Her application included fixes for several staging drivers. 

Two other Intel-mentored interns are in the mix: Hema Prathaban will be working with Jacob Pan on an Ivy Bridge temperature sensor driver, while Laura Mihaela Vasilescu will be working on Intel Ethernet drivers, mentored by Carolyn Wyborny and Anjali Singhai. Prathaban graduated in 2011 from KLN College of Engineering in India with a bachelor's degree in computer science. She has been a full-time mother for the last year, so the internship provides her a way to get back into the industry. Vasilescu is a master's student at the University of Politehnica of Bucharest, Romania and is also the student president of [ROSEdu](<http://www.rosedu.org/>), an organization for Romanian open source education. Both did a number of patches; Prathaban in the staging tree (including fixing a [bug](<https://groups.google.com/forum/#!msg/opw-kernel/m3GhH2r-GFc/E_UQKlqPTIoJ>) in one driver) and Vasilescu in Intel Ethernet drivers. 

#### Getting started

As with many budding kernel developers, most of the applicants' patches were to various staging drivers. There was a short application window as the kernel portion didn't get announced until a little under two weeks before the deadline. But that didn't seem to slow anything down as there were 41 applicants for the internships, with eighteen submitting patches and eleven having those patches accepted into the mainline. 

That level of interest—and success—is partly attributable to a [first patch tutorial](<http://sarah.thesharps.us/2013/05/23/%EF%BB%BF%EF%BB%BFopw-update/>) that she wrote, Sharp said. The tutorial helps anyone get started with kernel development from a fresh Ubuntu 12.04 install. It looks at setting up email, getting a kernel tree, using git, building the kernel, creating a patch, and more. The success was also due to strong applicants and mentors that were ""patient and encouraging"", she said. 

The kernel OPW program was mentioned multiple times at the recently held Linux Foundation conferences in Japan as a helpful step toward making the gender balance of kernel developers better represent the world we live in (as Dirk Hohndel put it). It is also nice to see the geographical diversity of the interns, with Asia, Europe, and North America all represented. Hopefully South America, Africa, and Oceania will appear in follow-on rounds of the program—Antarctica may not make the list for some time to come. 

Another round of the OPW, including kernel internships, is planned for January through March 2014 (with application deadlines in December). The program is seeking more interested projects, mentors, and financial backers for the internships. While there are certainly critics of these types of efforts, they have so far proved to be both popular and effective. Other experiments, using different parameters or criteria, are definitely welcome, but reaching out and making an effort to bring more women into the free-software fold is something that will hopefully be with us for some time—until that hoped-for day when it isn't needed at all anymore.
