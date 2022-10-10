---
title: Cisco 4500-X downgrade/upgrade procedure
date: 2015-12-27T10:31:08+00:00
author: burningnode
layout: post
categories: post
tags:
  - 4500X
  - catalyst
  - cisco
  - downgrade
  - ios
  - network
  - switch
  - upgrade
draft: false
---

Here is the upgrade/downgrade procedure for the 4500-X. For the record.

First upload the IOS archive (.bin) on the 4500-X. This can be done through tftp, scp or USB. My favorite way now is USB (the flash drive must be formatted in FAT32).

```
copy usb0:/xxxxxx.bin bootflash:/xxxxxx.bin
```

Then change the boot variable and write the configuration

```
(config)#boot system flash bootflash:/xxxxxx.bin
```

Make sure the configuration register is set to 2102 (take startup config and boot variable into account)

```
(config)#config-register 0x2102
(config)#end
#write
#reload
```

_Updated on 10/12/2016._