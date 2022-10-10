---
title: 'Microsoft Windows Server licensing scared me again'
date: 2013-03-30T11:42:26+00:00
author: burningnode
layout: post
categories: post
tags:
  - licensing
  - microsoft
  - server
  - windows
  - windows server 2012
draft: false
---
Two days ago, I was performing migration work. We used to run a few Windows Server 2012 Datacenter Evaluation Edition and the licenses were to expire in the next few days.

I asked for official commercial licenses and I received them a few days later. Then I just wanted to replace the evaluation edition key with an official one&#8230;. and when I got this message, I facepalmed&#8230;

![http://www.brngnd.com/images/2013/03/win2012-errorlicence.jpg](/win2012-errorlicence.jpg)

Yes Datacenter Evaluation edition is actually different from Datacenter edition. Well&#8230; Happily I found a work around on technet that I wanted to share with you :

_The following operation will require the server to restart_  
&#8211; Open Windows CLI (cmd.exe)  
&#8211; Type the following commands

```
DISM /online /Get-CurrentEdition
DISM /online /Get-TargetEditions
DISM /online /Set-Edition:editionID /ProductKey:XXXX-XXXX-XXXX-XXXX /AcceptEula
```

editionID: ServerDatacenter  
XXXX-XXXX-XXXX-XXXX: new product key

&#8211; Restart the server and let it work until it is finished

I finally obtained a commercial Datacenter Edition.


[http://technet.microsoft.com/en-us/library/jj574204.aspx](http://technet.microsoft.com/en-us/library/jj574204.aspx)