---
title: 'Challenge pkt300 - DEFCON 16 explained'
date: 2013-05-05T18:16:51+00:00
author: burningnode
layout: post
categories: courses
tags:
  - challenge
  - defcon16
  - pcap
  - security
  - ssl
draft: false
---

This challenge was given at DEFCON 16 and was a bit tricky I must say. I took approximately three hours to do it but it was mainly due to the fact that I was unable to find the good .pem file directly. The google dork given as hint sent a few results back in 2008, but now, there are hundreds of results! I went in circle asking what did I do wrong, and finally I decided to look for and try more keys. I finally found the good one on this [http://twistedmatrix.com/trac/browser/trunk/twisted/test/server.pem](http://twistedmatrix.com/trac/browser/trunk/twisted/test/server.pem).

With this key I was able to decipher the messages. Here's the method:

[slides](http://www.brngnd.com/images/2013/06/explained-pkt300-defcon16.pdf)

I always appreciate to work on these topics, as they make me learn new things and/or keep my knowledge up to date.

Thanks Luc.



[Challenge files here](http://stalkr.net/files/defcon/18/quals/packet300/)  
[Challenge from one hour to the next in 2008 (Spanish)](http://dumacx.blogspot.fr/2010/05/el-fin-de-semana-pasado-entre-el-21-y.html)