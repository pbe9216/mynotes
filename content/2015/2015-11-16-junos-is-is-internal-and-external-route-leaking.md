---
title: 'JunOS: IS-IS internal and external route leaking'
date: 2015-11-16T19:12:33+00:00
author: burningnode
layout: post
categories: post
tags:
  - is-is
  - juniper
  - junos
  - network
  - route leaking
  - routing
draft: false
---

We saw in the previous JunOS IS-IS post that two mechanism allowed the full connectivity among the level 1 and 2 areas:  
&#8211; the default route generated on L1 devices  
&#8211; the full view on L2 devices permitted by the automatic leak of L1 internal routes inside L2 area (done at L1L2 intermediate systems)  

I want all my routers to have same routing table, to have a full view of the backbone prefixes.  

For that purpose, we need to enable L2 routes to leak into L1 areas.  

![Junos lab backbone ipv4 is-is topology](http://www.brngnd.com/images/2015/11/junos-bbone-lab-isis-topology.png")

Let&#8217;s configure L2 to L1 route leaking.  
We are going to amend the configuration the two L1L2 routers (see them as similar to OSPF ABRs).  

First, define a policy to select which routes are leaking:  

R1, R4

```
policy-options {
    policy-statement L2-to-L1 {
        term level2 {
            from {
                protocol isis;
                level 2;
            }
            to level 1;
            then accept;
        }
    }
}

protocols {
    isis {
        export L2-to-L1;
```

Once, done, all the routes are leaked, we can check on R8

```
root@JOSR8# run show route protocol isis
inet.0: 18 destinations, 18 routes (18 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0.0.0.0/0          *[IS-IS/15] 02:16:57, metric 10
                    > to 10.10.1.1 via em0.0
10.10.2.0/24       *[IS-IS/15] 02:36:31, metric 20
                    > to 10.10.1.1 via em0.0
10.10.3.0/24       *[IS-IS/15] 02:36:31, metric 20
                    > to 10.10.1.1 via em0.0
10.10.4.0/24       *[IS-IS/18] 00:09:33, metric 30
                    > to 10.10.1.1 via em0.0
10.10.5.0/24       *[IS-IS/18] 00:09:33, metric 40
                    > to 10.10.1.1 via em0.0
10.10.6.0/24       *[IS-IS/18] 00:09:33, metric 30
                    > to 10.10.1.1 via em0.0
10.10.7.0/24       *[IS-IS/18] 00:09:33, metric 40
                    > to 10.10.1.1 via em0.0
10.10.9.0/24       *[IS-IS/18] 00:09:33, metric 30
                    > to 10.10.1.1 via em0.0
10.10.25.1/32      *[IS-IS/15] 02:36:31, metric 10
                    > to 10.10.1.1 via em0.0
10.10.25.2/32      *[IS-IS/18] 00:09:33, metric 20
                    > to 10.10.1.1 via em0.0
10.10.25.3/32      *[IS-IS/18] 00:09:33, metric 30
                    > to 10.10.1.1 via em0.0
10.10.25.4/32      *[IS-IS/18] 00:09:33, metric 30
                    > to 10.10.1.1 via em0.0
10.10.25.5/32      *[IS-IS/18] 00:09:33, metric 40
                    > to 10.10.1.1 via em0.0
10.10.25.6/32      *[IS-IS/18] 00:09:33, metric 20
                    > to 10.10.1.1 via em0.0
10.10.25.7/32      *[IS-IS/18] 00:09:33, metric 30
                    > to 10.10.1.1 via em0.0

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)

```

Now, let&#8217;s check what happen with an external route in a L1 area.

I create the following static route on R5 _(discard keyword is the juniper&#8217;s way of creating a null route)_

```
root@JOSR5# edit routing-options
root@JOSR5# set static route 10.10.24.0/24 discard
```

I can check in the RT: 

```
root@JOSR5# run show route 10.10.24.0

inet.0: 19 destinations, 19 routes (19 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.10.24.0/24      *[Static/5] 00:14:11
                      Discard

```

Let&#8217;s now create the appropriate route policy

```
policy-options {
    policy-statement EXTERNAL {
        term STATIC {
            from protocol static;
            to protocol isis;
            then accept;
        }
    }
}
protocols {
    isis {
        export EXTERNAL;
```

And check the result on R4 after committing

```
root@JOSR4# run show route protocol isis 10.10.24.0

inet.0: 20 destinations, 20 routes (20 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.10.24.0/24      *[IS-IS/15] 00:00:33, metric 10
                    > to 10.10.7.5 via em1.0

```

The route is well received.  
The same check is performed on R2 (L2 device)

```
root@JOSR2# run show route protocol isis 10.10.24.0

inet.0: 20 destinations, 20 routes (20 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.10.24.0/24      *[IS-IS/18] 00:04:43, metric 30
                      to 10.10.9.3 via em1.0
                      to 10.10.9.6 via em1.0
                    > to 10.10.4.3 via em2.0

```

And again same check on R8

```
root@JOSR8# run show route protocol isis 10.10.24.0

inet.0: 19 destinations, 19 routes (19 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.10.24.0/24      *[IS-IS/18] 00:10:15, metric 40
                    > to 10.10.1.1 via em0.0

```

The route is propagated.  
Well, the normal IS-IS operation is not to propagate IS-IS L1 external routes to L2 areas and other L1 areas. This is true but with the wide-metrics used in the network IS-IS has no way to determine whether the route is Internal or External as it was the case with standard metrics. Hence, the system considers all routes to be Internal, and thus, pass them from an area to another. If the wide metrics were not used, a route leaking policy from Level 1 to Level 2 would have been necessary on the R4 L1L2 router.   

We can check on how the route is seens in link state database 

```
root@JOSR2# run show isis database JOSR4.00-00 detail
IS-IS level 1 link-state database:

IS-IS level 2 link-state database:

JOSR4.00-00 Sequence: 0x11, Checksum: 0x6cb8, Lifetime: 747 secs
   IS neighbor: JOSR4.03                      Metric:       10
   IS neighbor: JOSR6.03                      Metric:       10
   IP prefix: 10.10.5.0/24                    Metric:       10 Internal Up
   IP prefix: 10.10.6.0/24                    Metric:       10 Internal Up
   IP prefix: 10.10.7.0/24                    Metric:       10 Internal Up
   IP prefix: 10.10.24.0/24                   Metric:       10 Internal Up
   IP prefix: 10.10.25.4/32                   Metric:        0 Internal Up
   IP prefix: 10.10.25.5/32                   Metric:       10 Internal Up

```

Note that the default configuration on Juniper routers is to advertise both narrow and wide metrics.  
With the wide-metrics-only keyword, the TLV 2, 128 and 130 are no more attached in link state PDUs. The remaining TLVs 22 and 135 make no distinction between Internal and External.  