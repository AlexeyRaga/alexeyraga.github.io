---
title: (Re)Building my own NAS
layout: post
tags:
  - NAS
  - ZFS
---

I have my Windows Home Server for years now. Its primary function is storage, but it also serves as a playground/testground for my own code, as a torrent client and for some other small things that I occasionally need.<br>
After all these years I have decided to decommission my "old" server and to get another one instead.

The reasons I am not happy about the current one are:

- **It runs Windows Home Server 2011, 32bit edition**. Well, it is Windows and I don't do much with Windows these days. It is 32bit so I can't have >3GB RAM and 3GB isn't much or even enough. There is no way to "just upgrade" Windows Home Server from 32bit to 64bit. And this OS is discontinued by Microsoft so there is no future in it anyway.
- **It is very slow**. Not only because of RAM, but because I've built it this way: I wanted a very quiet machine (so almost no fans) and I didn't need much power back then. Its network is slow as well (I attempted to fix it several times, but I could't figure out the cause).
- **It started making noise**. I am picky about noise. I know I could invest in just replacing noisy parts, but since I am not happy about my server anyway I decided not to.

With that in mind my core requirements are still the same + I have got some extra ones:

- **It must be quiet**. Did I mention how I feel about the noise? (I hate my freezer).
- **Bottleneck should be my network** and not the server or file system performance. And it should use network efficiently.
- **It should be able to serve media content** so installing things like [Plex](https://www.plex.tv/) for real-time transcoding should not be a problem.
- **I want to run stuff** securely and easily. To me it means containers (Docker) and, maybe, VMs.

### TL;DR - FreeNAS, ZFS

I've chosen to build a new server around [ZFS](https://en.wikipedia.org/wiki/ZFS) because it is awesome. I won't be explaining its awesomeness here, google if you are interested, but it **IS** awesome.<br>
As an OS I've decided to use [FreeNAS](http://freenas.org/) because it is designed for NAS, is built around ZFS and it has an awesome and helpful community (forums, documentation, etc.).

### What I could have done instead

I was thinking about updating my home server for a while evaluating different options. Here are some of them:

**Just buy a NAS**

I could buy something from Synology or QNAP. Initially I wanted to do just that, but then I decided that I don't really like to trust a hardware RAID controller (not that I have any experience). Reading about these NAS solution I also found out that it could be really painful to extend the storage or to upgrade its capacity. Some people also complain about the noise. Price is also a factor if I want to plan ahead: 10-12 disks models are pretty expensive. Plus I am not sure if they could nicely run my containers/VMs. And yes, the backup is a big question.

**Upgrade to Windows Storage Spaces**

This means having Windows again. This is not a problem in itself, but most of the stuff that I am doing now isn't Windows. Also, according to reviews, Storage Spaces and ReFS are slow. Plus ReFS really looks like an attempt to implement parts of ZFS ("was not invented here" syndrome?). And, of course, I don't have a Windows Server license and I am not willing to buy one.

**Install Ubuntu + ZFS**

This one looks interesting. FreeNAS is built on FreeBSD which I haven't seen for more than a decade. Most of the servers I work with have Ubuntu (others have different Linux). And it looks like [ZFS is now fully supported by Ubuntu](https://wiki.ubuntu.com/ZFS).

But it looks like ZFS is a relatively new thing for Ubuntu. The community around it seems to be smaller and there is less material (which is mostly references FreeBSD or Solaris). Choosing this option would also mean more work for me setting and configuring things up, compare to FreeNAS.

### So FreeNAS + ZFS

First thing first, after reading through ZFS use cases and scenarios I had to come up with my new server hardware. This is the part I don't like most - finding out nuances of different CPUs, motherboards, etc. and figuring out how it all will work together.

Fortunately, FreeNAS forums were extremely useful (again) and have already come up with a list of "dos" and "don'ts" with really compelling reasons for each of them.<br>
They also maintain a list of "recommended" hardware which has been used and tested by the members of the community. With this my "job" became much easier: understand requirements, understand reasons for these requirements, look at the examples.

In the end I came up with the following list:

**Motherboard**<br>
[Supermicro MBD-X10SL7](https://www.supermicro.com/products/motherboard/Xeon/C220/X10SL7-F.cfm).

- has enough SATA plugs for me to expand (I plan to use just 6-7 initially).
- has IPMI support which is great because I don't have a monitor at home. IPMI allows connecting to the motherboard through the network to manage BIOS settings as well as managing the OS.
- has SATA DOM (Disk on Module) support.

**Memory**<br>
**4 x Samsung M391B1G73QH0-YK0 8GB DDR3-1600 1.35V 2Rx8 ECC Un-Buffer LP PB-Free**.<br>
Yes, it is 32GB total which may seem overkill for NAS. But ZFS loves RAM much more than other systems. Also my new server is not only NAS, here are my VMs, media (Plex), etc.

Yes, it is ECC. I have some valuable daya (~6TB now) and it is growing. And I'd hate if something bad happened to it. I read multiple times about how [ECC is important for ZFS](http://jrs-s.net/2015/02/03/will-zfs-and-non-ecc-ram-kill-your-data/). Then I looked up the price and realised that today is not 2004 (or when I built a computer last time) and that it is very reasonable these days.

So it is 32GB and it is ECC, and it is NOT overkill, get over it ;)

**CPU**<br>
**Intel Xeon E3-1231V3 Haswell 3.4 GHz**.<br>
It is powerful so will be enough for all the things I want it to do: video transcoding, running my VMs, etc. Maybe it is a bit of an overkill, will see. But I also wanted to plan for the future a bit because last time I chose a "just enough for NAS" CPU and ended up being unable to do what I wanted.

**System Disk**<br>
[16GB SATA DOM MV 3ME](https://www.memorydepot.com/ssd/ssddetails.html?prodid=DESMV-A28D06RC1QCF)<br>
FreeNAS doesn't use data pool for keeping the OS, and the OS drive is mostly immutable. Many use just flash drives to boot up FreeNAS (which is less reliable), others use small SSD disks. SATA DOM seems to be an easy and convenient choice: just plug it into the SATA plug on the board and that's it.<br>
This was one of the recommended options on a FreeNAS forums and I liked it.<br>
Could it be an overkill? Could I get a small SSD instead? Would USB flash be more convenient? Maybe it doesn't matter when it is just $25.

**Data Disks**<br>
I have a bunch of 1TB-3TB disks in my old (current) NAS which I plan to reuse and maybe replace later. I also bought 2x4TB WD Red disks (which are designed for NAS).<br>
Because NAS is not a backup, I still want a separate backup copy of the important parts my data (which is most of it) so I haven't decided whether these 2 new disks should go to the main pool or to the backup one.

**Power Supply**<br>
[Seasonic G-550](https://seasonic.com/product/g-550/).<br>
I wanted a good and a very quiet PSU so I ended up looking at 80+ Gold certified models and comparing reviews about noise. I did not end up looking at 80+ Platinum because they have a significant jump in price while not as much of a jump in specs.<br>
It still has one (promised to be very quiet) fan and I am still in doubt about whether I needed to go with fanless models or not, but I will see.

**Case**<br>
[Fractal Design Define R5](http://www.fractal-design.com/home/product/cases/define-series/define-r5-black)<br>
I spent _much_ more time looking up cases than anything else. I needed a _very quiet_ case that could hold at least 8 HDDs (maybe 2 more in the future) and could still have a good airflow.<br>
After reading and watching lots of reviews I made my choice.<br>
R5 comes with system fans without PWM and I decided to try it first this and see if I actually need to them to be replaced with PWM-enabled ones. If so then I get something like [Noctua NF-P14s REDUX](http://noctua.at/en/products/product-line-redux/nf-p14s-redux-1200-pwm).

### What is next

I got most of the parts from this list delivered (well, everything except the system disk and the case) so the next step will be putting it all together, testing the hardware and installing FreeNAS. I will write about my experience once I am through it.
