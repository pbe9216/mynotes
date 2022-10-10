---
title: Azure Web App VNET integration networking
date: 2021-04-05T10:46:11+00:00
author: burningnode
layout: post
categories: post
tags:
  - network
  - azure
  - cloud
  - microsoft
  - virtual
  - vnet
  - webapp
  - vnet integration
  - security
draft: false
---

In the previous notes I talked about service endpoints and private links, this note will be about Web app VNET integration feature. The name sounds similar, but the solution is quite different.

In a nutshell, VNET integration allows a WebApp to access resources in your VNET without having them publicly available. It does not make your webapp reachable from your VNET. That is the main point.
So, traffic is meant to be only from Web App to the VNET and not the way around. 

Web App service relies on different App Service hosting plans, which are in fact containers. VNET integration function will provide a direct interface between your container hosting your web app to your VNET.

![Azure VNET integration](/azure-vnet-integ.jpg)

This function is useful if you want to access in a more private fashion some private resources like backends, databases or azure services connected and reachable in your VNET.
To setup VNET integration a dedicated and empty subnet will be required in your VNET for that purpose.

If you connect to your container, you can check on the initial routing table and verify it cannot communicate with your private VM IP.
```
root@14b76fa5b17c:/home# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.16.2.1      0.0.0.0         UG    0      0        0 eth0
172.16.2.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0

root@14b76fa5b17c:/home# ping 10.0.0.4
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
^C
--- 10.0.0.4 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 15ms
```

With VNET integration activated, here is what happened in the container, as you can see a new interface is created and RFC1918 prefixes + your VNET prefix are routed through that interface: 
```
root@9c716033e7b0:/home# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         169.254.129.1   0.0.0.0         UG    0      0        0 eth0
10.0.0.0        169.254.254.1   255.255.255.0   UG    0      0        0 vnet00sfpui6f
10.0.0.0        169.254.254.1   255.0.0.0       UG    0      0        0 vnet00sfpui6f
169.254.129.0   0.0.0.0         255.255.255.0   U     0      0        0 eth0
169.254.254.0   0.0.0.0         255.255.255.0   U     0      0        0 vnet00sfpui6f
172.16.0.0      169.254.254.1   255.240.0.0     UG    0      0        0 vnet00sfpui6f
192.168.0.0     169.254.254.1   255.255.0.0     UG    0      0        0 vnet00sfpui6f
```

You can check the ping and this time it works. 
```
root@9c716033e7b0:/home#  ping 10.0.0.4
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
64 bytes from 10.0.0.4: icmp_seq=1 ttl=63 time=2.26 ms
64 bytes from 10.0.0.4: icmp_seq=2 ttl=63 time=2.55 ms
64 bytes from 10.0.0.4: icmp_seq=3 ttl=63 time=2.00 ms
^C
--- 10.0.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 2.003/2.271/2.549/0.223 ms
```

A simple PHP code can be used to check if a fictitious backend is available via port 80
```
root@14b76fa5b17c:/home# cat /var/www/html/wwwroot/index.php
<?php
$fp = fsockopen("10.0.0.4", 80, $errno, $errstr, 30);
if($fp) {
    echo("Connection OK");
} else {
    echo("Connection NOK");
}
?>
```

On the VM side it is possible to run a tcpdump to observe the traffic coming from the webapp container:
```
contoso@rg1vm1:~$ sudo tcpdump -i any "icmp" -nn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
22:42:47.067173 IP 10.0.0.190 > 10.0.0.4: ICMP echo request, id 54, seq 1, length 64
22:42:47.067209 IP 10.0.0.4 > 10.0.0.190: ICMP echo reply, id 54, seq 1, length 64
22:42:48.068413 IP 10.0.0.190 > 10.0.0.4: ICMP echo request, id 54, seq 2, length 64
22:42:48.068442 IP 10.0.0.4 > 10.0.0.190: ICMP echo reply, id 54, seq 2, length 64
22:42:49.073790 IP 10.0.0.190 > 10.0.0.4: ICMP echo request, id 54, seq 3, length 64
22:42:49.073820 IP 10.0.0.4 > 10.0.0.190: ICMP echo reply, id 54, seq 3, length 64
22:42:50.078169 IP 10.0.0.190 > 10.0.0.4: ICMP echo request, id 54, seq 4, length 64
22:42:50.078200 IP 10.0.0.4 > 10.0.0.190: ICMP echo reply, id 54, seq 4, length 64

contoso@rg1vm1:~$ sudo tcpdump -i any "port 80" -nn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
22:45:47.523480 IP 10.0.0.190.47392 > 10.0.0.4.80: Flags [S], seq 1217983486, win 64240, options [mss 1418,sackOK,TS val 597696570 ecr 0,nop,wscale 7], length 0
22:45:47.523529 IP 10.0.0.4.80 > 10.0.0.190.47392: Flags [S.], seq 2548114986, ack 1217983487, win 65160, options [mss 1460,sackOK,TS val 2206057966 ecr 597696570,nop,wscale 7], length 0
22:45:47.525514 IP 10.0.0.190.47392 > 10.0.0.4.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 597696573 ecr 2206057966], length 0
22:45:47.525549 IP 10.0.0.190.47392 > 10.0.0.4.80: Flags [F.], seq 1, ack 1, win 502, options [nop,nop,TS val 597696573 ecr 2206057966], length 0
22:45:47.525648 IP 10.0.0.4.80 > 10.0.0.190.47392: Flags [F.], seq 1, ack 2, win 510, options [nop,nop,TS val 2206057968 ecr 597696573], length 0
22:45:47.527358 IP 10.0.0.190.47392 > 10.0.0.4.80: Flags [.], ack 2, win 502, options [nop,nop,TS val 597696575 ecr 2206057968], length 0
```
As you can see, an IP has been used in the VNET integration subnet.

We can verify that the opposite direction is not working:
```
contoso@rg1vm1:~$ curl http://10.0.0.190:80/
curl: (7) Failed to connect to 10.0.0.190 port 80: Connection refused
contoso@rg1vm1:~$ curl -k https://10.0.0.190/
curl: (7) Failed to connect to 10.0.0.190 port 443: Connection refused
```
As we can see, traffic is not forwarded, I cannot access the web page from there.

## Resources

[https://docs.microsoft.com/en-us/azure/app-service/web-sites-integrate-with-vnet](https://docs.microsoft.com/en-us/azure/app-service/web-sites-integrate-with-vnet)







