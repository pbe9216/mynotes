---
title: 'Linux Workshop #2 The Linux firewall'
date: 2013-02-15T21:23:26+00:00
author: burningnode
layout: post
categories: courses
tags:
  - courses
  - firewall
  - linux
draft: false
---

Linux Workshop #2 &#8211; Linux Firewall

[Linux Firewall](/Linux_Workshop_2-Linux_Firewall_v2.pdf)


Sample iptables script:

```
#!/bin/bash
### BEGIN INIT INFO
# Provides: firewall.sh
# Required-Start:
# X-Start-Before:
# Default-Start: 2 3 4 5
# Required-Stop:
# Default-Stop: 0 1 6
# Short-Description: Start the ip(6)tables firewall.
# Description: Start the ip(6)tables firewall.
### END INIT INFO

case $1 in
start)
# Clear everything
iptables -F
# Deny all
iptables -t filter -P INPUT DROP
iptables -t filter -P FORWARD DROP
iptables -t filter -P OUTPUT DROP
# Allow ICMP
iptables -t filter -A INPUT -p icmp -j ACCEPT
iptables -t filter -A OUTPUT -p icmp -j ACCEPT
# Allow SSH
iptables -t filter -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -t filter -A OUTPUT -p tcp --dport 22 -j ACCEPT
# Allow HTTP and DNS in output to enable package installation
iptables -t filter -A OUTPUT -p tcp --dport 80 -j ACCEPT
iptables -t filter -A OUTPUT -p udp --dport 53 -j ACCEPT
# Connections
iptables -t filter -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -t filter -A OUTPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
;;
stop)
# Clear everything and allow all
iptables -F
iptables -t filter -P INPUT ACCEPT
iptables -t filter -P FORWARD ACCEPT
iptables -t filter -P OUTPUT ACCEPT
;;
esac
```

