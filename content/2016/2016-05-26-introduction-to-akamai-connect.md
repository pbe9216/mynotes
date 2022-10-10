---
title: Introduction to Akamai Connect
date: 2016-05-26T23:17:51+00:00
author: burningnode
layout: post
categories: post
draft: false
---

We will discuss what is Akamai Connect (also known as AKC) in the following article. Akamai Connect is a &#8220;plugin&#8221; (this is how I define it) for the Wide Area Application Services solution (WAAS). It is conjointly developed between Cisco and Akamai.  
Akamai is a leading provider for content distribution and caching technologies. They develops and operates an advanced platform that leverages multiple technologies : content distribution and replication, advanced caching mechanism, advanced delivery mechanisms (Sure Route IP). More information can be found on Akamai websites at the end of this article.

AKC is a Cisco product, sold and supported by Cisco, but the software and the rules inside are developed by Akamai. The objective of the Akamai Connect solution is to provide advanced caching features to the WAAS device to improve the last mile communication quality / performance (speed). The WAAS device do basically byte caching and deduplication (DRE) but not a HTTP protocol / context aware caching. The AKC is integrated in the WAAS but has to be activated with a license purchase. It can **only** be managed through the WAAS Central Manager (you cannot setup an Akamai Connect product without the CM, because the license registration is only performed from there).  

There are two types of scenarios supported by Akamai Connect. Because AKC is network agnostic, it can be used to cache internal content (served through MPLS WAN line) as well as external websites (internet link). In the current release (6.1 at this time) AKC and WAAS more generally can only optimize clear-text content in single-sided mode (one WAAS), thus HTTPS sites cannot be cached. The only scenario where SSL protected content can be accelerated and cached is in dual-sided mode where the connection is decrypted/crypted by the WAAS. This can only be done with internal content. This implies the WAAS to be able to decipher the communication (with private keys). In the 6.2 release (May 2016), single-sided decryption has been added to the feature set, it is called SmartSSL (more information can be found in the [6.2 release notes](http://www.cisco.com/c/en/us/td/docs/app_ntwk_services/waas/waas/v621/note/waas621xrn.html)).

[WAAS-dual-single-sided](/WAAS-dual-single-sided.jpg)

The following caching technologies are supported with Akamai Connect product (excerpt from a Cisco Live presentation):  
![akc-caching-methods](/akc-caching-methods.png)

The Connected Cache option consists in treating specifically the content if it has been served from the Akamai CDN (WAAS find this out by observing HTTP headers). In that case specific rules apply and content stays potentially longer in the cache. 

What must be clear is that the AKC is not an Akamai Edge server. An edge server is part of the Akamai CDN and is connected to their overlay network. AKC has no intent to be an edge server and does not connect to the global CDN network. The set of rules between AKC and Akamai CDN are completely different. The OTT rules are solely used in the Akamai Connect solutions. The Akamai rules for general content distribution are only used on the edge servers on the other hand. The OTT rulebase is generally served from a specific Akamai server but sometimes can be from an edge.  

Akamai Connect is activated from the CM. Go to Home > Admin > Licenses > Akamai Connect, then choose a file and upload.  

![AKC-licenses](/AKC-licenses.png)

To manage AKC settings for the WAAS devices, go to the Device Groups (best practice is to use groups instead of single units).  
Device Groups > Configure > Caching > Akamai Connect

![AKC-configuration-general](/AKC-configuration-1b.png)

Two tabs are available: Cache settings and Cache prepositioning.

First let&#8217;s talk about the Cache Settings. In order to activate AKC, the &#8220;Enable Akamai Connect&#8221; checkbox must be activated.  
The &#8220;Edit Settings&#8221; pane lets you activate OTT cache and Connected Cache features and the &#8220;Advanced Cache Settings&#8221; pane allows you to adjust the caching policies. In this menu you can specify a certain caching technique (or no caching at all) for a specific website.

![AKC-configuration-cache](/AKC-configuration-2.png)

The checkbox &#8220;Use an HTTP Proxy&#8221; can be used to control how the AKC connects to the internet (recall that AKC needs to download the OTT rules from Akamai). An external proxy (with no authentication) or the CM can be used. The CWS checkbox is used when a cloud proxy is used by the clients. In order to prevent cache content to be served to the wrong person this option will force a refresh each time the cache is asked, this lower the performances but ensure security consistency. 

The prepositioning settings allow to pre-cache a website. The goal behind this technique is to avoid the first slow connection to retrieve and cache the content (normal caching process).  
The configuration pane permits to schedule a caching job. It can be used to cache any types of content.

![AKC-configuration-prepositioning](/AKC-configuration-3.png)

Those options are interesting but before engaging yourself in this path I really recommend to test this system. Akamai Connect may or may not answer to your need depending on the content or on what your current solution does. You need to evaluate what is the benefit for your specific case to justify buying it and what are the limitations you will face (e.g.: the maximum cacheable object size). If your target websites are 10% cacheable by AKC, then the performance improvement will not be seen by the end clients. Asking your Cisco sales rep for a test phase is a good start as he or she will put you in touch with technical engineers from both companies.  

The impact of deploying this solution (WAAS on top of the ISR4K, Akamai Connect, AppNav, etc&#8230;) can also be questioned as it clearly adds a layer of complexity in your wide area network, so, does it really worth it ? does the improvements brought make a clear difference? does the improvements justify the complexity? Those aspects should not be underestimated and should be discussed!  

Links:  
[http://www.cisco.com/c/en/us/solutions/enterprise-networks/intelligent-wan-akamai/index.html](http://www.cisco.com/c/en/us/solutions/enterprise-networks/intelligent-wan-akamai/index.html)     
[https://www.akamai.com/](https://www.akamai.com/)   
[https://developer.akamai.com/stuff/](https://developer.akamai.com/stuff/)     
[https://developer.akamai.com/stuff/Optimization/SureRoute.html](https://developer.akamai.com/stuff/Optimization/SureRoute.html)    
[https://www.akamai.com/fr/fr/solutions/products/cloud-networking/cisco-intelligent-wan-with-akamai-connect.jsp](https://www.akamai.com/fr/fr/solutions/products/cloud-networking/cisco-intelligent-wan-with-akamai-connect.jsp)   