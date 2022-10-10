---
title: Time based ACLs
date: 2013-12-20T21:02:06+00:00
author: burningnode
layout: post
categories: post
tags:
  - acl
  - cisco
  - ios
  - network
  - security
  - time
  - time based acl
draft: false
---

Time based Access Control Lists are ACLs that embbed another parameter, guess what, time.  
This an example of what it is possible to do with time based ACLs. Although, these ACLs could be used together with other mechanisms such as QoS MQC and ZBF.

- Any traffic must be denied except backup traffic (port 100) on weekends  
- HTTP, HTTPS must be allowed every weekdays from noon to 2pm  
- FTP, SSH must be allowed on weekdays from 8am to 8pm  
- Telnet will be authorized from January 1 2014 5am, to February 1 2014 6am

First check that your router got a synchronized clock.

```R1#sh clock
*00:10:29.011 UTC Fri Mar 1 2002

R1#sh ntp status
%NTP is not enabled.</pre>
```

This is clearly not the case in my example! In real life, every network devices should be synchronized with a trusted time source.



Based on these four assertions, we can create four time ranges for our ACLs.

```
time-range BACKUP
 periodic weekend 0:00 to 23:59
!
time-range FTP-SSH
 periodic weekdays 8:00 to 20:00
!
time-range HTTP-S
 periodic weekdays 12:00 to 14:00
!
time-range TELNET
 absolute start 05:00 01 January 2014 end 06:00 01 February 2014
```


Create and apply the ACL as any other ACL  
The time ranges are bound using the time-range attribute in the ACL&#8217;s statements

```
ip access-list extended TIMEBASEDACL
 permit tcp any any eq 100 time-range BACKUP
 permit tcp any any eq www time-range HTTP-S
 permit tcp any any eq 443 time-range HTTP-S
 permit tcp any any eq ftp time-range FTP-SSH
 permit tcp any any eq ftp-data time-range FTP-SSH
 permit tcp any any eq 22 time-range FTP-SSH
 permit tcp any any eq telnet time-range TELNET
 deny   ip any any log
```

```
interface FastEthernet0/0
 ip access-group TIMEBASEDACL in
```



Verification

```
R1#sh time-range
time-range entry: BACKUP (inactive)
   periodic weekend 0:00 to 23:59
   used in: IP ACL entry
time-range entry: FTP-SSH (inactive)
   periodic weekdays 8:00 to 20:00
   used in: IP ACL entry
   used in: IP ACL entry
   used in: IP ACL entry
time-range entry: HTTP-S (inactive)
   periodic weekdays 12:00 to 14:00
   used in: IP ACL entry
   used in: IP ACL entry
time-range entry: TELNET (inactive)
   absolute start 05:00 01 January 2014 end 06:00 01 February 2014
   used in: IP ACL entry
R1#sh ip access-lists
Extended IP access list TIMEBASEDACL
    10 permit tcp any any eq 100 time-range BACKUP (inactive)
    20 permit tcp any any eq www time-range HTTP-S (inactive)
    30 permit tcp any any eq 443 time-range HTTP-S (inactive)
    40 permit tcp any any eq ftp time-range FTP-SSH (inactive)
    50 permit tcp any any eq ftp-data time-range FTP-SSH (inactive)
    60 permit tcp any any eq 22 time-range FTP-SSH (inactive)
    70 permit tcp any any eq telnet time-range TELNET (inactive)
    80 deny ip any any log
```

HTH

