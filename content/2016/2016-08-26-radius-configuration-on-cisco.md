---
title: RADIUS configuration on Cisco
date: 2016-08-26T21:40:02+00:00
author: burningnode
layout: post
categories: post
draft: false
---
Recently I had to work on a RADIUS configuration for Cisco network devices. The goal was to have a central authentication service, which is kind of the basis nowadays. Every devices in the network must use the corporate RADIUS server to authenticate the administrators. It simplifies the account management as the RADIUS server can rely on a pre-existing directory server for the user database (Active Directory or any LDAP speaking directory for some of them), the privileges are dynamically passed to the remote devices and the actions are logged, so we can see who was connected to which equipment at a precise time.  

As I was used to the old way to write those configurations snippets, I learned the new commands this time. You will find below a sample configuration:  

First configure the RADIUS server groups  
&#8211; timeout value defines how long the switch waits to resend a packet  
&#8211; automate-test enables automatic service checks on the authentication and authorization ports

```
radius server SERV1
 address ipv4 1.1.1.1 auth-port 1812 acct-port 1813
 timeout 10
 automate-tester username radius probe-on
 key cisco
!
radius server SERV2
 address ipv4 1.1.1.2 auth-port 1812 acct-port 1813
 timeout 10
 automate-tester username radius probe-on
 key cisco

```

Then call those servers inside the group configuration  
&#8211; Configure a group of servers (call the servers defined earlier, define the source address, define the load-sharing method)  
&#8211; Define the order of authentication and authorization sources evaluation, in this case the RADIUS-FARM first and secondly the local database

```
aaa new-model
!
aaa group server radius RADIUS-FARM
 server name SERV1
 server name SERV2
 ip radius source-interface Vlan2
 load-balance method least-outstanding
!
aaa authentication login RADIUS-LOGIN group RADIUS-FARM local
aaa authorization exec RADIUS-AUTHORIZATION group RADIUS-FARM local
aaa accounting exec RADIUS-ACCOUNTING start-stop group RADIUS-FARM
```

Then, configure the vty lines on the device to use the AAA methods defined previously:  

```
line vty 0 4
 authorization exec RADIUS-AUTHORIZATION
 accounting exec RADIUS-ACCOUNTING
 logging synchronous
 login authentication RADIUS-LOGIN
 transport input ssh
 transport output ssh
```

In case of failure, during the authentication the first server will be evaluated and if it does not answer, the second one will be evaluated.  
If none of the above answer, the authentication will fall back to the local database as stated in the aaa configuration.  

For troubleshooting purposes, you can use the following command to check servers liveliness and authentication requests  

```
#show aaa servers

RADIUS: id 3, priority 1, host 1.1.1.1, auth-port 1812, acct-port 1813
     State: current UP, duration 1927725s, previous duration 220s
     Dead: total time 777s, count 4
     Quarantined: No
     Authen: request 119, timeouts 49, failover 0, retransmission 37
             Response: accept 64, reject 6, challenge 0
             Response: unexpected 0, server error 0, incorrect 0, time 93ms
             Transaction: success 70, failure 12
             Throttled: transaction 0, timeout 0, failure 0
     Author: request 0, timeouts 0, failover 0, retransmission 0
             Response: accept 0, reject 0, challenge 0
             Response: unexpected 0, server error 0, incorrect 0, time 0ms
             Transaction: success 0, failure 0
             Throttled: transaction 0, timeout 0, failure 0
     Account: request 166, timeouts 52, failover 1, retransmission 40
             Request: start 58, interim 0, stop 57
             Response: start 54, interim 0, stop 56
             Response: unexpected 0, server error 0, incorrect 0, time 62ms
             Transaction: success 114, failure 12
             Throttled: transaction 0, timeout 0, failure 0
     Elapsed time since counters last cleared: 3w4d1h39m
     Estimated Outstanding Access Transactions: 0
     Estimated Outstanding Accounting Transactions: 0
     Estimated Throttled Access Transactions: 0
     Estimated Throttled Accounting Transactions: 0
     Maximum Throttled Transactions: access 0, accounting 0
     Requests per minute past 24 hours:
             high - 0 hours, 2 minutes ago: 2
             low  - 1 hours, 24 minutes ago: 0
             average: 0

RADIUS: id 4, priority 2, host 1.1.1.2, auth-port 1812, acct-port 1813
     State: current UP, duration 1927724s, previous duration 200s
     Dead: total time 705s, count 4
     Quarantined: No
     Authen: request 78, timeouts 45, failover 5, retransmission 33
             Response: accept 29, reject 4, challenge 0
             Response: unexpected 0, server error 0, incorrect 0, time 110ms
             Transaction: success 33, failure 12
             Throttled: transaction 0, timeout 0, failure 0
     Author: request 0, timeouts 0, failover 0, retransmission 0
             Response: accept 0, reject 0, challenge 0
             Response: unexpected 0, server error 0, incorrect 0, time 0ms
             Transaction: success 0, failure 0
             Throttled: transaction 0, timeout 0, failure 0
     Account: request 95, timeouts 44, failover 5, retransmission 33
             Request: start 27, interim 0, stop 24
             Response: start 23, interim 0, stop 24
             Response: unexpected 0, server error 0, incorrect 0, time 50ms
             Transaction: success 51, failure 11
             Throttled: transaction 0, timeout 0, failure 0
     Elapsed time since counters last cleared: 3w4d1h36m
     Estimated Outstanding Access Transactions: 0
     Estimated Outstanding Accounting Transactions: 0
     Estimated Throttled Access Transactions: 0
     Estimated Throttled Accounting Transactions: 0
     Maximum Throttled Transactions: access 0, accounting 0
     Requests per minute past 24 hours:
             high - 1 hours, 21 minutes ago: 0
             low  - 1 hours, 21 minutes ago: 0
             average: 0

```

HTH.