---
title: Introduction to Policy Based Routing
date: 2013-01-26T21:51:04+00:00
author: burningnode
layout: post
categories: courses
tags:
  - cisco
  - ios
  - junos
  - pbr
  - policy based routing
  - routing
draft: false
---

Introduction to Policy Based Routing (PBR).

**Slides**

[Policy Based Routing-Introduction](/Policy-Based-Routing-Introduction.pdf)


**Labs**

_These topologies are not representing real scenarios._  
Client VM: [Ubuntu](http://www.ubuntu.com/download/desktop)  
Server VM: [TurnKey Linux OrangeHRM](http://www.turnkeylinux.org/orangehrm)  
Cisco Routers: 3725 adventreprise  12.4(18)


**Case #1 &#8211; PBR based on destination port**

Configs: [pbrcase1](http://www.brngnd.com/images/2013/01/pbrcase1.zip)

![PBR case1](/case1.jpg)


**Case #2 &#8211; PBR based on source addresses**

Configs: [pbrcase2](http://www.brngnd.com/images/2013/01/pbrcase2.zip)

![PBR case2](/case2.jpg)



**Case #3 &#8211; Load sharing based on traffic type with redundant gateways**

[PBR case3](/pbrcase3.zip)


**Case #4 &#8211; PBR with NBAR**

_Be careful with NBAR in production environments !_

Configs: [pbrcase4](http://www.brngnd.com/images/2013/01/pbrcase4.zip)

![PBR case4](/case4.jpg)



[http://www.cisco.com/en/US/products/hw/routers/ps359/products_tech_note09186a00800fc176.shtml#markinboundhacks](http://www.cisco.com/en/US/products/hw/routers/ps359/products_tech_note09186a00800fc176.shtml#markinboundhacks)

[http://blog.ine.com/2008/11/04/using-nbar-for-http-url-filtering/](http://blog.ine.com/2008/11/04/using-nbar-for-http-url-filtering/)

[http://www.cisco.com/en/US/docs/ios/12_2/qos/configuration/guide/qcfpbr.html](http://www.cisco.com/en/US/docs/ios/12_2/qos/configuration/guide/qcfpbr.html)





**Case #5 &#8211; PBR on JunOS**

Topology: same as Case #1

Config:

```
Interfaces
{
 em0 {
        unit 0 {
            family inet {
                address 10.10.10.1/24;
            }
        }
    }
 em1 {
        unit 0 {
            family inet {
                address 10.10.20.1/24;
            }
        }
    }
}

routing-instances {
    ISP1 {
        instance-type forwarding;
        routing-options {
            static {
                route 0.0.0.0/0 next-hop 1.1.1.2;
            }
        }
    }
    ISP2 {
        instance-type forwarding;
        routing-options {
            static {
                route 0.0.0.0/0 next-hop 2.2.2.2;
            }
        }
    }
}

protocols {
    ospf {
        rib-group MERGE-CONNECTED;
        area 0.0.0.0 {
            interface em0;
        }
    }
}
routing-options {
    interface-routes {
        rib-group inet MERGE-CONNECTED;
    }
    rib-groups {
        MERGE-CONNECTED {
            import-rib [ inet.0 ISP1.inet.0 ISP2.inet.0 ];
        }
    }
}

policy-options {
    prefix-list LAN-A {
        10.10.10.0/24;
    }
    prefix-list LAN-B {
        10.10.20.0/24;
    }
    prefix-list NET {
        0.0.0.0/0;
    }
}

firewall {
    family inet {
        filter SOURCE-ROUTING-SELECTION {
            term FROM-LAN-1 {
                from {
                    source-prefix-list {
                        LAN-A;
                    }
                    destination-prefix-list {
                        NET;
                    }
                }
                then {
                    routing-instance ISP1;
                }
            }
            term FROM-LAN-B {
                from {
                    prefix-list {
                        LAN-B;
                    }
                }
                then {
                    routing-instance ISP2;
                }
            }
            term DEFAULT {
                then accept;
            }
        }
    }
}
```



[http://blog.inetsix.net/2012/06/policy-based-routing-with-junos/](http://blog.inetsix.net/2012/06/policy-based-routing-with-junos/)

[http://jncie.files.wordpress.com/2008/09/350136_filter-based-forwarding.pdf](http://jncie.files.wordpress.com/2008/09/350136_filter-based-forwarding.pdf)

[http://www.juniper.net/techpubs/en_US/nsm2012.1/topics/concept/security-service-firewall-screenos-policy-based-routing-overview.html](http://www.juniper.net/techpubs/en_US/nsm2012.1/topics/concept/security-service-firewall-screenos-policy-based-routing-overview.html)

