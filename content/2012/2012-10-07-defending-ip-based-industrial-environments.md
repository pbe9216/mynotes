---
title: Defending IP-based industrial environments
date: 2012-10-07T09:18:25+00:00
author: burningnode
layout: archives
categories: ["post", "archived"]
tags:
  - industrial
  - network
  - security
draft: false
---
Recent events have shown that industrial production systems are more vulnerable than we initially thought. In my opinion the main cause of this, is the convergence of industrial networks and computer IP-based networks.

There has been a few types of networks until today:  
- Separated networks  
- Networks with a convergence point  
- Mixed environments

This convergence has brought the risks and the inherent vulnerabilities of IP-based computer networks into the industrial world. Because these networks were initially separated, industrial control systems (ICS) have never been built with security in mind, and thus, are vulnerable. With enough financial and human means organizations can easily develop viruses or worms to compromise this kind of systems.

The problem is that these systems support critical parts of our today&#8217;s infrastructures: power grids, nuclear plants, urban traffic systems, industrial productions systems&#8230; Successful attacks against such infrastructures could result in very serious consequences.

In addition, securing industrial environments is not an easy task. The constraint is that these networks are now tied to computer applications (e.g. ERPs, production monitoring systems, tracking systems&#8230;). And these applications need to be connected to both industrial systems and local networks.

In this article, I will detail some of the possible countermeasures to maintain security in these environments.

Reminder: this article only contains ideas and I cannot be held responsible for anything happening in your networks!

**Motivations**

First let us have a look to the motivations that could lead to a direct attack against industrial control systems. Because they ensure very important and critical functions they are obvious targets, and with network convergence that I talked about just before, it is easier to get to them and to cause damages.

There could be various reasons:  
- a competitor that want to slow down your production  
- a country that want to attack another (cyber warfare)  
- terrorist and criminal organizations that want to cause damage or make profit  
- activists that want to compromise specific systems (e.g. to show their vulnerabilities)

These threats a very real and every security officer must take them in account when defining corporate security policies

**Counter #1: Network Isolation**

The first countermeasure is network isolation. Network isolation is achieved thanks to Virtual LAN technology. This enables virtual networks to run on top of the physical wires so you can easily segment your L2/L3 domain. After that you can apply filtering rules on your VLANs through ACLs (Access Control Lists) or even firewall rules (which are more precise).

![Network isolation with VLANs](/vlans.jpg)

**Counter #2: Traffic sanitization**

Once separated from your LAN, the next step is to make sure that traffic entering your industrial VLAN is clean. This could be done using two methods: installing an intrusion detection and prevention system and/or an antivirus appliance.

**Using IDPS**

The benefits of such systems are to detect and take proactive measures (IPS) against network attacks. This is achieved using pattern matching mechanisms: all the traffic is analyzed and compared to a signature database. This database should be, of course, regularly updated.

IDS (Intrusion Detection System) are not set up inline and can only send alerts but cannot take any proactive measures. IPS are working as inline devices and are, de facto, able to detect and block suspicious traffic. Keep in mind that a false positive with an IPS can provoke network issues (blocking legitimate traffic).

The pros:  
- Network protection for well-known attacks  
- Could take proactive measures

The cons:  
- Depending on the hardware it could slow down the traffic (wire-speed)  
- Sensitive to false-positive / false-negative  
- Evasion techniques exists

**Using AV appliances**

Antivirus appliances are similar to IDPS in how they work: pattern matching operations take place between the traffic flowing in the network and a database of worms signatures. The aim is to block traffic associated to known worms and viruses from entering in the restricted area.  
These appliances could be physical or virtual (virtual machines) and are manufactured by a few security companies like: Barracuda, Trend, McAfee, Symantec.

![Traffic sanitization](/traffic-sanitization.jpg)

**Counter #3: Remote inspection**

Antivirus scanning is also a problem in industrial environments because generally we cannot have a good level of interaction with the systems: the systems could exotic (and there is no way to access it through conventional means) or the systems could be old and you cannot do whatever you want. For example, old windows systems like NT4 or 98 are still used to control industrial production lines, just because of old pieces of software that cannot run on newer platforms. These kinds of systems are very vulnerable due to their operating systems and scanning them is a challenge, because no antivirus can run on them.  
However a solution can be found to solve this problem: perform remote antivirus scan. On the operating system you need to share the workstation hard drive and then perform an antivirus with a dedicated server. With some scripting work you can create a fully automated solution that will scan your production stations without running particular software on them.  
However be careful when scanning your production workstations. Generally these workstations are equipped with old and low performances hardware pieces and an intense scan could create latency on the host. Remote scanning could also take a lot network bandwidth, so you must choose the most appropriate time.

![Remote Antimalware Scan](/remote-scan.jpg)

**Counter #4: Physical security**

In order to minimize the number of attack vectors, you should prevent all removable devices and unauthorized people to interact with your critical workstations.

e.g., on host :  
- disable USB ports and CD readers,  
- secure your BIOS if possible,  
- lock your workstation to prevent someone opening it.

Additional door and access security systems should be also set up to control access to your facilities:  
- RFID  
- Biometrics  
- CCTV  
- and more...

**Counter #5: Employee awareness and periodical controls**

As always, the best countermeasure is the employees&#8217; awareness towards security. Keep them updated of security alerts and educate them with basic security skills and you will avoid great pain.

A good example is this nuclear plant that was compromised by the worm Stuxnet. They had a fully isolated and secured network; however someone brought the worm unintentionally with a USB key&#8230;

Think about it.

**Conclusion**

The moral is:  
- segment and isolate your networks
- implement filtering appliances to monitor, clean and restrict the traffic  
- find a way to scan your workstations  
- implement physical countermeasures  
- train your employees regularly and keep them updated with security bulletins.  
- security teams must stay aware

By the way, do not forget that:  
- Antivirus provide protection only for well-known worms and viruses, thus could be easily bypassed,  
- There are evasion techniques to bypass network filtering,  
- Isolation could be compromised if not properly secured (VLAN hopping),  
- Employees could be a threat

I hope this has been informative for you.

**Links**

- [http://www.us-cert.gov/control_systems/ics-cert/](http://www.us-cert.gov/control_systems/ics-cert/")
- [http://csrc.nist.gov/publications/nistpubs/800-82/SP800-82-final.pdf](http://csrc.nist.gov/publications/nistpubs/800-82/SP800-82-final.pdf) 
- [http://en.wikipedia.org/wiki/Industrial_Control_System](http://en.wikipedia.org/wiki/Industrial_Control_System)
- [http://en.wikipedia.org/wiki/SCADA](http://en.wikipedia.org/wiki/SCADA)
- [en.wikipedia.org/wiki/Stuxnet](http://en.wikipedia.org/wiki/Stuxnet)
- [http://www.cisco.com/en/US/prod/collateral/vpndevc/ps5729/ps5713/ps4077/ips_industrial_control_protection.pdf](http://www.cisco.com/en/US/prod/collateral/vpndevc/ps5729/ps5713/ps4077/ips_industrial_control_protection.pdf)
- [http://www.cisco.com/web/about/security/intelligence/protecting_ics_networks_with_cisco_ips.html](http://www.cisco.com/web/about/security/intelligence/protecting_ics_networks_with_cisco_ips.html")  
- [http://www.cisco.com/web/strategy/manufacturing/ettf_overview.html](http://www.cisco.com/web/strategy/manufacturing/ettf_overview.html)
- [http://www.cisco.com/web/strategy/docs/manufacturing/industrial_ethernet.pdf](http://www.cisco.com/web/strategy/docs/manufacturing/industrial_ethernet.pdf)
