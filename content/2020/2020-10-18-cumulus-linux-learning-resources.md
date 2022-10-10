---
title: Cumulus Linux learning resources
date: 2020-10-18T08:44:08+00:00
author: burningnode
layout: post
categories: post
tags:
  - network
  - cumulus linux
  - linux
  - cconp
  - CCONP
  - resources
  - training
  - nvidia
draft: false
---

Here, I summed up some of the resources I used to gain knowledge on Cumulus Linux (now part of Nvidia group, with Mellanox) and pass CCONP certification exam.

First of all, about the exam itself. It is only delivered remotely through ProctorU platform ([https://www.proctoru.com/](https://www.proctoru.com/)). Due to Cumulus acquisition by Nvidia, some of the resources and websites are currently moved and merged with Mellanox ones. The way to access learning resources and purchase the exam is now through the [Mellanox academy](https://academy.mellanox.com/en/). 

The overall exam experience with ProctorU was smooth and good. As with all remotely proctored certification you'll need an empty and quiet room to take it. I personnally took it from a meeting room at the office. 

Regarding the knowledge required to pass the certification, you need to have a good understanding of common networking technologies, plus the cumulus / linux based networking sauce on top. Having a good background in network engineering I focused on the specific things introduced in this linux based network operating system.

You'll find some of the resources I went through or used / still use today:
- [Cumulus Linux website](https://docs.cumulusnetworks.com/), very good documentation, acurate most of the time :-), it contains detailed explanations, CLI snippets and also standard configuration examples (e.g.: VXLAN EVPN leaf spine)
- [Cumulus Linux slack](https://slack.cumulusnetworks.com/), very helpful community regrouping a lot of good engineers and also a quick way to get in touch with Cumulus folks. Clearly great initiative (Note that you have something similar around FRR project).
- [Oreilly - EVPN in the datacenter](https://cumulusnetworks.com/lp/evpn-data-center-oreilly/), author: Dinesh G. Dutt,
- [Oreilly - BGP in the datacenter](https://cumulusnetworks.com/learn/resources/guides/bgp-datacenter), author: Dinesh G. Dutt,
- [Linux Networking 101](https://www.actualtechmedia.com/wp-content/uploads/2017/12/CUMULUS-NETWORKS-Linux101.pdf), author: David M. Davis, ActualTech Media,
- [CCONP Study Guide](https://cumulusnetworks.com/learn/resources/guides/cconp-exam-study-guide), author: Craig Tompkins and maybe others
- a bit outdated due to the CL version used, but still a nice introduction, David Bombal's [Udemy course](https://www.udemy.com/course/cumulus-linux-fundamentals-plus-ansible-automation/) 
- the [GoldenTurtle ](https://gitlab.com/cumulus-consulting/goldenturtle) and [cumulus-consulting](https://gitlab.com/cumulus-consulting/) gitlab repos, containing config, for easy prototyping


I am lucky enough to touch switches running CL on a day to day basis, but I also spent quite some time labbing and testing topologies using the [Cumulus VX (virtual image)](https://cumulusnetworks.com/products/cumulus-vx/). It's a great way to gain knowledge and nearly all features are there. Of course, you'll not find hardware related intricacies but still it's truly valuable. In addition some CL engineers created and maintain the [topology converter](https://gitlab.com/cumulus-consulting/tools/topology_converter) that can be used to create lab scenarios based on a .dot file. You should be able to create a management stack (jump host with external access, management switch, dhcp activated) automatically, it's great, but I faced some issues in that last part. I never investigated it more as I made it work on the fly, may be I did something wrong...

Before knowing the existence of that tool and how to use it, I created my own conversion script that take a .grapml file as input. You can draw graphically a topology using [yEd desktop editor](https://www.yworks.com/products/yed) and use the script to spin up the vagrant based virtual lab . It's basic and subject to improvements, but you'll find it [there](https://gitlab.com/burningnode/myvlabmanager) (recently moved to gitlab).

Last but not least, you can find great resources on those blogs (not necessarily focused on Cumulus Linux):
- [ipSpace / Ivan Pepelnjak](https://blog.ipspace.net/), one of the most valuable - documented, in-depth and mythbusting articles, very informative videos and speakers,...
- Follow and watch Pete Lumbis and Dinesh Dutt (among others) webinars, talks and write ups
- [packetpushers](https://packetpushers.net/) website and podcast,
- and many more that I cannot remember presently...

Hope this helps,
