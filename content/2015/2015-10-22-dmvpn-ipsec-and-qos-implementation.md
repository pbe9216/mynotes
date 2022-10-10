---
title: 'DMVPN: IPSec and QoS implementation'
date: 2015-10-22T14:38:07+00:00
author: burningnode
layout: post
categories: post
tags:
  - dmvpn
  - ipsec
  - networks
  - nhrp
  - qos
  - routing
draft: false
---

The next step is the IPsec implementation. DMVPN can run with GRE only or with GRE+IPSec which is generally the preferred way as tunnels are built on top of the internet (unsecure network). Even over a MPLS VPN solution provided by your ISP going for an IPsec DMVPN makes sense to enforce communication security. After this, we will also go through the QoS implementation on DMVPN and the tricky part of spoke to spoke QoS.  

Three steps are required to activate IPsec on hub and spokes routers:  

1. Configure an ISAKMP policy and an ISAKMP key:  

```
crypto isakmp policy 10
 encryption aes
 authentication pre-share
 group 5

crypto isakmp key dmvpncloud1 address 0.0.0.0 0.0.0.0

```

The “group” defined is the algorithm used for IKE key exchange, 2 is Diffie-Hellman 1024bits. You may have to adapt to the lower end router for the supported algorithms.  

```
R1(config-isakmp)#group ?
  1   Diffie-Hellman group 1 (768 bit)
  14  Diffie-Hellman group 14 (2048 bit)
  15  Diffie-Hellman group 15 (3072 bit)
  16  Diffie-Hellman group 16 (4096 bit)
  19  Diffie-Hellman group 19 (256 bit ecp)
  2   Diffie-Hellman group 2 (1024 bit)
  20  Diffie-Hellman group 20 (384 bit ecp)
  24  Diffie-Hellman group 24 (2048 bit, 256 bit subgroup)
  5   Diffie-Hellman group 5 (1536 bit)

R6(config-isakmp)#group ?
  1  Diffie-Hellman group 1
  2  Diffie-Hellman group 2
  5  Diffie-Hellman group 5
```

2. Configure the transform-set:  

```
crypto ipsec transform-set TSET esp-aes esp-sha-hmac
 mode transport
```

3. Set up an IPsec profile:  

```
crypto ipsec profile DMVPN
 set security-association lifetime seconds 300
 set transform-set TSET
 set pfs group5
```

4. Once defined, push the profile on the tunnel interface:

```
interface tunnel 0 
 tunnel protection ipsec profile DMVPN
```

IPSec is a protocol suite designed to secure IP protocol. IPsec is compliant with the CIA triad as it provides confidentiality (encryption), integrity (data validation and anti-replay mechanisms) and authentication (talkers authentication). For this purpose, IPsec relies on various cryptographic components that are described in numerous RFCs (20+). IPsec basically relies on AH and ESP protocols as well as SA protocols/algorithms. Depending on the needs IPsec can work in two modes: transport and tunnel.  
AH (Authentication Header, RFC 2402) provides authentication, integrity and anti-replay mechanism. Note that AH only does not provide encryption of the IP packet but security and identification of the talkers.  
ESP (Encapsulation Security Payload, RFC 2406) provides, on the other end, authentication, integrity, encryption and anti-replay mechanism. It is possible to use both AH and ESP in an IPsec communication, but we tend to limit the encapsulation overhead.  

ESP can be used in transport mode, where only the IP payload will be encrypted leaving the IP header (addresses, options) unencrypted. It can also be used in tunnel mode where it completely encapsulates the IP packet. This is true to a certain extend, but in DMVPN the first encapsulation is GRE and then is ciphered using IPsec parameters, so no difference will be seen between mode transport and mode tunnel actually (see capture below). Knowing that, transport mode is recommended to save a few bytes in the header anyway, and tunnel mode can be set up in particular cases were different devices would perform DMVPN and IPSec encryption.  

Integrity and anti-replay mechanisms are brought in IPsec with the use of hash (one way cryptographic function) and HMAC (seals).  

In IPv6, IPsec is integrated as an “extension” to the base header.  

SA, stands for security associations; they are IPsec security parameters agreed between two endpoints for a given communication (and stored in SA Database). Each side maintain a SA, and each side parameters must match in order to establish reliable communication. SA are identified with the destination endpoint address, the security protocol(s) and the Security Parameter Index (SPI), a unique 32-bit value derived from the two previous parameters.  

For the purpose of this example we use pre-shared key preconfigured on the router, but IPsec can use different key distribution mechanism and can rely on a PKI / x509 certificates. The exchange mechanism is detailed below in the ISAKMP and IKE sections.  

IKE (Internet Key Exchange) is the protocol used to create the SAs. IKE is available in version 1 and version 2 and is referenced in the following RFCs: RFC2409 (IKE), RFC4109 (IKEv1 Algorithms) RFC5996 (IKEv2).  

ISAKMP means Internet Security Association and Key Management Protocol and provides a framework for authentication and key exchange for Security Associations establishment. ISAKMP works with other protocol to build SAs. ISAKMP defines procedures and packet formats to establish, negotiate, modify and delete SA. ISAKMP uses UDP and port 500. ISAKMP is implemented on numerous operating systems like IOS, OpenBSD and Linux. ISAKMP is defined in RFC2408, and is used as base for IKEv1. Other protocols relies on ISAKMP.  

For IKEv1 there are two phases to bring the IPsec tunnel.  
&#8211; Phase 1: a first tunnel is created which protects the future IKE negotiation message for the IPsec tunnel. The goal of phase 1 is two provide a secure channel for a small amount of data. Phase 1 can be implemented in main mode where peers identity is protected or aggressive mode where it is not. Aggressive mode skip steps in the SA negotiation making it faster and lighter in term of processing.   
&#8211; Phase 2: a second tunnel is created with the parameters exchanged previously. IPsec tunnel is built within the IKE SA and is said to be “quick mode”.  

IKEv2 is a rework of IKE intended to define more precisely what was expected from the protocol and its behavior. This in order to eliminate the different implementation of IKEv1 that historically created problems between vendors. IKEv2 is simpler, faster, generates less overhead, consumes less bandwidth and provide notably NAT traversal feature as well as better protection against denial of services attacks (DoS can be achieved by initiating a lot of fake DH sessions, affecting routers CPU or hardware modules ; this is now prevented using a cookie exchange mechanism prior to the DH session creation).  
IKEv2 has different phases respectively called: IKE\_SA\_INIT, IKE\_AUTH, Create\_CHILD_SA ([http://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/115936-understanding-ikev2-packet-exch-debug.html](http://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/115936-understanding-ikev2-packet-exch-debug.html)).

Last but not least, PFS (Perfect Forward Secrecy) has been added to the configuration. It is used to renew the DH key for each DH session, thus preventing any interception of the key by an attacker.

Here is the configuration for the four routers participating in the DMVPN cloud:

```
crypto isakmp policy 10
 encryption aes
 authentication pre-share
 group 5

crypto isakmp key dmvpncloud1 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TSET esp-aes esp-sha-hmac
 mode transport

crypto ipsec profile DMVPN
 set security-association lifetime seconds 300
 set transform-set TSET
 set pfs group5

interface tunnel 0 
 tunnel protection ipsec profile DMVPN

```

The configuration is the same, and this one of the advantages of DMVPN. Endpoints and SA are directly handled and negotiated within the DMVPN software piece. Recall that will site to site IPsec tunnels we had to configure a crypto-map with the crypto-ACL (to capture encrypted traffic) and defined the remote endpoint. This is no more necessary with DMVPN has it is handled automatically, which necessary for spoke to spoke traffic.  

We can check the status of the IPsec tunnels and the DMVPN:  

```
R1#sh dmvpn detail
Legend: Attrb --&gt; S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket
        # Ent --&gt; Number of NHRP entries with same NBMA peer
        NHS Status: E --&gt; Expecting Replies, R --&gt; Responding, W --&gt; Waiting
        UpDn Time --&gt; Up or Down Time for a Tunnel
==========================================================================

Interface Tunnel0 is up/up, Addr. is 192.168.1.1, VRF ""
   Tunnel Src./Dest. addr: 1.1.1.1/MGRE, Tunnel VRF ""
   Protocol/Transport: "multi-GRE/IP", Protect "DMVPN"
   Interface State Control: Disabled
Type:Hub, Total NBMA Peers (v4/v6): 3

# Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb    Target Network
----- --------------- --------------- ----- -------- ----- -----------------
    1        6.6.6.6     192.168.1.6    UP 00:00:18    D     192.168.1.6/32

    1        7.7.7.7     192.168.1.7    UP 00:00:16    D     192.168.1.7/32

    1        8.8.8.8     192.168.1.8    UP 00:00:17    D     192.168.1.8/32



Crypto Session Details:
--------------------------------------------------------------------------------

Interface: Tunnel0
Session: [0x67620188]
  IKEv1 SA: local 1.1.1.1/500 remote 6.6.6.6/500 Active
          Capabilities:(none) connid:1004 lifetime:23:59:39
  IKEv1 SA: local 1.1.1.1/500 remote 6.6.6.6/500 Inactive
          Capabilities:(none) connid:1003 lifetime:0
  Crypto Session Status: UP-ACTIVE
  fvrf: (none), Phase1_id: 6.6.6.6
  IPSEC FLOW: permit 47 host 1.1.1.1 host 6.6.6.6
        Active SAs: 2, origin: crypto map
        Inbound:  #pkts dec'ed 22 drop 0 life (KB/Sec) 4413303/280
        Outbound: #pkts enc'ed 29 drop 0 life (KB/Sec) 4413301/280
   Outbound SPI : 0x47C7658D, transform : esp-aes esp-sha-hmac
    Socket State: Open


```

```
R1#sh crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
1.1.1.1         7.7.7.7         QM_IDLE           1006 ACTIVE
1.1.1.1         8.8.8.8         QM_IDLE           1005 ACTIVE
1.1.1.1         6.6.6.6         QM_IDLE           1004 ACTIVE

```

ISAKMP SA is mounted, QM stands for Quick Mode, which is IKE phase 2 (IPsec)

```
R1#sh crypto ipsec sa peer 6.6.6.6

interface: Tunnel0
    Crypto map tag: Tunnel0-head-0, local addr 1.1.1.1

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (1.1.1.1/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (6.6.6.6/255.255.255.255/47/0)
   current_peer 6.6.6.6 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 13, #pkts encrypt: 13, #pkts digest: 13
    #pkts decaps: 16, #pkts decrypt: 16, #pkts verify: 16
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 1.1.1.1, remote crypto endpt.: 6.6.6.6
     path mtu 1514, ip mtu 1514, ip mtu idb Loopback0
     current outbound spi: 0x311F0A49(824117833)
     PFS (Y/N): Y, DH group: group5

     inbound esp sas:
      spi: 0x80DB5650(2161858128)
        transform: esp-aes esp-sha-hmac ,
        in use settings ={Transport, }
        conn id: 91, flow_id: SW:91, sibling_flags 80000006, crypto map: Tunnel0-head-0
        sa timing: remaining key lifetime (k/sec): (4520501/287)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE

     inbound ah sas:

     inbound pcp sas:

     outbound esp sas:
      spi: 0x311F0A49(824117833)
        transform: esp-aes esp-sha-hmac ,
        in use settings ={Transport, }
        conn id: 92, flow_id: SW:92, sibling_flags 80000006, crypto map: Tunnel0-head-0
        sa timing: remaining key lifetime (k/sec): (4520501/287)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE

     outbound ah sas:

     outbound pcp sas:
R1#

```

Difference between transport and tunnel mode in capture are the overhead due to a field added in the header (length 214B vs 182B):  

Tunnel:  
![IPsec tunnel mode](/ipsec-tunnel.png)

Transport:  
![IPsec transport mode](/ipsec-transport.png)

Quality of Service can be implemented on IPsec tunnels. Actually two things can be done: mark the outer header (traffic exiting the interface, simply) and/or mark the encapsulated packet so the DSCP value will be treated somewhere else in the IP corporate network (qos pre-classify). This one of many example we can give. The first one can be useful when using VPNs across network with SLA such as MPLS grade solutions or even satellite solutions. The second one can be useful when an internet IPsec tunnel connect to a network with SLA (tunnel connecting to a MPLS cloud, tunnel connecting to a HQ enterprise network&#8230;).  

In our example our service provider offer 2 class of services: IMPORTANT (AF31) and BE (0), and our corporate network match AF21 to prioritize a business application. So we will have to do the following: external header set to AF31, and internal header set to AF21.  

/!\ In the rest of this document I have disabled tunnel protection to easily perform captures /!\  

```
ip access-list extended BIZ
 permit ip any 11.11.11.0 0.0.0.255
!
policy-map OUTER
 class BIZ
  set dscp af31
 class class-default
!
policy-map INNER
 class BIZ
  set dscp af21
!
interface Tunnel0
 service-policy output INNER
!
interface FastEthernet0/0
 service-policy output OUTER

```

Let’s try without qos pre-classify enable

![DMVPN QoS without preclassify](/dmvpn-qos1.png)

AF21 is pushed in both IP headers. Only INNER policy-map is applied.  
Now after qos-preclassify enable:

```
interface Tunnel0
 qos pre-classify
 service-policy output INNER

```

![DMVPN with QoS preclassify](/dmvpn-qos2.png)

Both tags are seen where they should be, OUTER policy-map is taken into account.  
The need for pre-classify command really depends on the case and implementation required. I recommend checking in the documentation and lab-ing it before using it or not (Some information can be found at: [http://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-policing/10106-qos-tunnel.html](http://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-policing/10106-qos-tunnel.html), [http://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/WAN\_and\_MAN/QoS_SRND/QoS-SRND-Book/IPSecQoS.html#pgfId-44642](http://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/WAN\_and\_MAN/QoS_SRND/QoS-SRND-Book/IPSecQoS.html#pgfId-44642). In my case I needed two different tags to be applied on inner and outer header with a classification based on IP address, pre-classification was needed.  

There are also ways to perform per-spoke QoS, but it is a more tricky part using EEM scripts. Examples were given in Cisco Live session BRKSEC-4054.  