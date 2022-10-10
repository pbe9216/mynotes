---
title: Routed design and MLS at the access layer
date: 2012-12-28T23:50:48+00:00
author: burningnode
layout: post
categories: post
tags:
  - architecture
  - ccda
  - cisco
  - design
  - network
  - switch blocks
draft: false
---
Recent design guidelines recommend to implement L3 links down to the access layer. A few months ago, I had to think about a network design evolution and I proposed two solutions: keep the existing L2 network that spans on the entire campus or setup a routed design with MLS at the access layer.

Traditional designs put the L2/L3 boundary at the distribution layer, which is generally the first hop. The access layer usually runs L2 switches because they do not deal with IP (L3). A design where you put the L2/L3 network boundary at the access layer require L3 interfaces and routing protocols. This is made possible using multilayer switches.

In my opinion routed designs and MLS at the access layer bring a lot of new features and possibilities in our networks. Here is a comparison between old and new architectures.

![L2 L3 boundary](/l2-l3-boundary1.jpg)

**L2 only access layer**

Advantages  
- ease of use, plug and play  
- convenient for support teams

Drawbacks  
- Does not scale  
- Chatty protocols  
- Errors / misconfiguration can affect the entire network  
- Convergence times are sometimes too important (L2+L3)  
- Cannot manipulate forwarding information easily  
- Waste of resources

![Entreprise Diagram](/enterprise-schema-op1.jpg)

**L2/L3 access layer**

Advantages  
- Cleaner network  
- Scale well  
- Improved resiliency  
- Improved convergence time  
- Possibility to manipulate routing information  
- Better use of existing resources

Drawbacks  
- Need to change the IP and VLAN plans  
- Harder IP subnetting work  
- Most of support teams are not trained on these technologies

![Entreprise diagram 2](/enterprise-schema-op2.jpg)

Now, you see what you can do 🙂  
Thanks for reading.

More information on:  
[http://www.cisco.com/en/US/docs/solutions/Enterprise/Campus/routed-ex.html](http://www.cisco.com/en/US/docs/solutions/Enterprise/Campus/routed-ex.html)