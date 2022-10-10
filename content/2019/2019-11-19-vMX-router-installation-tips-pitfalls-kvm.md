---
title: Juniper vMX router installation and troubleshooting in KVM
date: 2019-11-22T22:28:11+00:00
author: burningnode
layout: post
categories: post
tags:
  - network
  - vmx
  - lab
  - juniper
  - kvm
  - linux 
  - vpfe
  - vre
draft: false
---

Below you'll find a compilation of notes and errors encountered during the installation of the Juniper vMX router in KVM.
The vMX image provides a nice way to have a local lab, and it is fairly manageable with vmx script, virsh and maybe a script of your own ;-) 
It is also a solution aiming at service providers willing to have network function virtualization (NFV), thus this router can support high performance throughput.

## vMX router

First of all, you must be aware that the vMX router is actually made of two parts
- vRE / vCP : virtual routing engine, the control plane
- vPFE / vFP : virtual packet forwarding engine, the data plane

As this virtual router aims to be deployed in production environment you have different modes of installation to gain in performance with, for example, the direct access to the network card (SR-IOV) feature.
In our case we will stay with the standard installation and lite mode. The lite mode can be configured inside the VM directly.

## Preparing the system 

Ok, so you basically need KVM and libvirt installed.

Be sure to load the drivers for nested virtualisation ([https://wiki.ubuntu.com/kvm](https://wiki.ubuntu.com/kvm))
You can also go through this documentation before a production installation [https://www.juniper.net/documentation/en_US/vmx/topics/topic-map/vmx-installing-on-kvm.html#id-preparing-the-ubuntu-host-to-install-vmx](https://www.juniper.net/documentation/en_US/vmx/topics/topic-map/vmx-installing-on-kvm.html#id-preparing-the-ubuntu-host-to-install-vmx)

```
$ sudo modprobe kvm-intel
```

If you run into this error, it means that your Intel-VT / AMD-V virtualization options are disabled in the BIOS or not supported at all by your CPU.
```
FATAL: Error inserting kvm_intel (/lib/modules/2.6.20-15-generic/kernel/drivers/kvm/kvm-intel.ko): Operation not supported
Typing dmesg you may find the following at the end:-
```

Then you can retry.
To make this settings permanent, you can adjust those configuration files
```
screw@kvmhost:~/vmx$ sudo vim /etc/sysctl.conf
screw@kvmhost:~/vmx$ sudo vim /etc/default/qemu-kvm
```

## Starting the vMX 

### Create configuration files

The configuration file is a YAML file which can be broke down into a few parts: 

The configuration of the host and the links to the qcow2 images:
```
---
#Configuration
HOST:
        identifier : R2
        host-management-interface : ens33
        routing-engine-image : "/home/screw/vmx/images/junos-vmx-x86-64-17.2R1.13.qcow2"
        routing-engine-hdd : "/home/screw/vmx/images/vmxhdd.img"
        forwarding-engine-image : "/home/screw/vmx/images/vFPC-20170523.img"
---
```
The external bridge configuration useful to get access to the hosts externally :
```
#External bridge configuration
BRIDGES:
        - type  : external
          name  : br-ext
 ---
```

Then the vRE VM parameters, such as CPUs, RAM, console port, interfaces. 
This interface will be used to communicate with the vPFE.
```
#vRE VM parameters
CONTROL_PLANE:
        vcpus       : 1
        memory-mb   : 1024
        console_port: 8601

        interfaces  :
          - type      : static
                ipaddr    : 10.102.144.201
                macaddr   : "0A:00:DD:d8:4f:b8"

 ---
```

Then the vPFE VM parameters, with the interface, in the same IP subnet to allow communication between the two.
A specific bridge will be created.
```
#vPFE VM parameters
FORWARDING_PLANE:
        memory-mb   : 2048
        vcpus       : 3
        console_port: 8601
        device-type : virtio

        interfaces  :
          - type      : static
                ipaddr    : 10.102.144.41
                macaddr   : "0A:00:DE:4f:84:23"
 ---
```

Finally the interfaces configuration. These are the production interfaces you will use to run routing protocols and other stuff
```
 ---
#Interfaces
JUNOS_DEVICES:
   - interface : ge-0/0/0
     mac-address          : "02:06:0A:7b:84:50"
     description          : "ge-0/0/0 interface"
   - interface : ge-0/0/1
     mac-address          : "02:06:0A:4d:ec:ce"
     description          : "ge-0/0/1 interface"
```

### Links configuration files 

The links configuration file is documented here: [https://www.juniper.net/documentation/en_US/vmx14.1/topics/task/configuration/vmx-virtio-devices-binding.html](https://www.juniper.net/documentation/en_US/vmx14.1/topics/task/configuration/vmx-virtio-devices-binding.html)

The file is another YAML file with the following format (an example sits in config/vmx-junosdev.conf) : 
```
interfaces :

     - link_name  : vmx_link1
       mtu        : 1500
       endpoint_1 :
         - type        : junos_dev
           vm_name     : vmx1
           dev_name    : ge-0/0/0
       endpoint_2 :
         - type        : bridge_dev
           dev_name    : bridge1

     - link_name  : vmx_link2
       mtu        : 1500
       endpoint_1 :
         - type        : junos_dev
           vm_name     : vmx2
           dev_name    : ge-0/0/0
       endpoint_2 :
         - type        : bridge_dev
           dev_name    : bridge1

     - link_name  : vmx_link3
       endpoint_1 :
         - type        : junos_dev
           vm_name     : vmx1
```

### Launch instances

This will launch the first instance : 
You will repeat this step for each router.
```
sudo bash vmx.sh --install --cfg config/R1.conf
```

Then you can launch the connection script: 
```
sudo bash vmx.sh --bind-dev –-cfg config/links.conf
```


## Errors encountered while starting the vMX

### Bash, line not interpreted

Some lines are shown not to be interpreted.
This is because the shebang specify sh instead of bash.

You can either use the ```bash``` command to launch the script or replace ```#!/bin/sh``` by ```#!/bin/bash``` in the vmx.sh script.

### Huge pages

Generally, the error comes from the privileges of the user that executed the script. Run the script as root so it make the necessary verification regarding libvirt and hugepages.
If the error persist, re-run the script again.

Then, if it still fails, you can check that everything is properly configurer on your host:
```
pub@kvmhost:~/vmx$ cat /etc/default/qemu-kvm | grep HUGE
KVM_HUGEPAGES=1

pub@kvmhost:~/vmx$ cat /proc/meminfo | grep Huge
AnonHugePages:         0 kB
HugePages_Total:      44
HugePages_Free:       44
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:    1048576 kB

pub@kvmhost:~/vmx$ cat /etc/sysctl.conf | grep -i huge
# Allocate 256 HugePageTables (start with a low number but increas it before using it
vm.nr_hugepages = 8192

```

Useful links:
[https://forums.juniper.net/t5/vMX/VMX-install-fails-on-quot-setup-huge-pages-quot/td-p/309471](https://forums.juniper.net/t5/vMX/VMX-install-fails-on-quot-setup-huge-pages-quot/td-p/309471)
[https://help.ubuntu.com/community/KVM%20-%20Using%20Hugepages](https://help.ubuntu.com/community/KVM%20-%20Using%20Hugepages)

###  network 'br-ext' already exists

If this happens it means that you have an already br-ext bridge registered but not active.
It may be the results of an unsuccessful attempt to run the script or a previous setup.
In order to clean this, you can do it by running the following virsh commands: 

```
pub@kvmhost:~/vmx$ virsh

virsh # net-list
 Name                  State      Autostart Persistent
 ----------------------------------------------------------
 default              active      yes           yes

virsh # net-list --all
 Name                  State      Autostart Persistent
 ----------------------------------------------------------
 br-ext               inactive    no            yes
 default              active      yes           yes

virsh # net-undefine br-ext
Network br-ext has been undefined

virsh # net-list --all
 Name                  State      Autostart Persistent
 ----------------------------------------------------------
 default              active      yes           yes
```

If the network is active and you want to achieve the same results, you need to destroy it first with:
```
net-destroy
```

### Libvirt file creation issue

If you encounter the following error, install the corresponding python module: 
```
File "/home/nugraha/Documents/vmx/scripts/common/vmx_configure.py", line 9, in <module>
    import netifaces as ni
ImportError: No module named netifaces
```

Source is here : [https://forums.juniper.net/t5/vMX/vMX-Failed-Generate-libvirt-files/td-p/286736](https://forums.juniper.net/t5/vMX/vMX-Failed-Generate-libvirt-files/td-p/286736)

### panic: CPU0 does not support X87 or SSE: 1

You have to configure the cpu-mode as **host-passthrough** in the vRE xml.
```
vim build/vmx1/xml/vRE-generated.xml
  <cpu mode="host-passthrough">
```

Sources: 
[https://forums.juniper.net/t5/vMX/vMX-cannot-boot-up-staying-at-db-gt/td-p/305658](https://forums.juniper.net/t5/vMX/vMX-cannot-boot-up-staying-at-db-gt/td-p/305658)
[https://kb.juniper.net/InfoCenter/index?page=content&id=KB20635&actp=METADATA](https://kb.juniper.net/InfoCenter/index?page=content&id=KB20635&actp=METADATA)


### Failed to start domain

If you got into this error, you might need to check that you IPs, MACs or console ports are unique across your configuration files.
```
error : internal error: early end of file from monitor, possible problem: device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x8 -msg timestamp=on
2018-05-21T17:24:21.734512Z qemu-system-x86_64: -chardev socket,id=charserial0,host=127.0.0.1,port=8600,telnet,server,nowait: Failed to bind socket: Address already in use
```


### Deleted default bridge

If default br is missing or you have deleted it accidentally, then recreate it from this configuration found on github:
```
<network>
  <name>default</name>
  <bridge name="virbr0" />
  <forward/>
  <ip address="192.168.122.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.122.2" end="192.168.122.254" />
    </dhcp>
  </ip>
</network>
```

### VM management 

You can use the following commands to manage your lab:
```
virsh 
list
destroy
net-list --all
net-undefine
```

### Auto Image Upgrade: DHCP Client State Reset: fxp0.0

This is an automatic process based on DHCP for operating system upgrade on Juniper switches. 
You can disable it by entering the following commands in JunOS configuration mode:
```
delete chassis auto-image-upgrade
commit
```

### Root password

As always, you will be asked to change the root password on the JunOS box, here is the snippet: 
```
edit system
set root-authentication plain-text-password XX....XX
```

### Message from syslog

Message from syslogd@ at Apr  9 15:28:36  ... fpc0 Frame 8: sp = 0xffeeb978, pc = 0x807c415
Message from syslogd@ at Nov 11 09:07:28  ... fpc0 Scheduler Oinker

This log message kept bugging me.
The only mean I found to silence it, is to put a regex to prevent it from getting its way to the console.
```
edit system syslog user *
set match "!fpc"
```

### Linecard disovery and communication checks

This Juniper troubleshooting procedure is interesting: [https://www.juniper.net/documentation/en_US/vmx/topics/task/verification/vmx-vm-connection-troubleshooting-esxi.html](https://www.juniper.net/documentation/en_US/vmx/topics/task/verification/vmx-vm-connection-troubleshooting-esxi.html)

Two things seen there: troubleshoot communication between vFP and vCP using ping, and check linecard discovery by vCP.

To ping, you need to find the IPs on both vCP and vFP. It can be done with ```show interfaces terse``` and ```ifconfig``` on vFP.
Then the following ping command is used:
``` 
root> ping 128.0.0.16 routing-instance __juniper_private1__
```

To check the line card discovery : ```show chassis fpc``` looking for linecard starting in slot 0 and ```show interfaces terse``` looking for ge-0/x/x interfaces.
If not two command are recommanded to restart the corresponding processes: 
```
request chassis fpc slot 0 restart
```

and if it fails showing ```FPC is in transition```:
```
restart chassis-control
```

### CPU usage / Configure lite-mode 

If you have an high CPU usage, you might need to change the performance mode to lite.

```
root# edit chassis fpc 0 
root# set lite-mode
```

Source:
[https://forums.juniper.net/t5/vMX/Juniper-VMX-bad-cpu-usage-using-lite-mode-in-kvm-compared-to/td-p/318927](https://forums.juniper.net/t5/vMX/Juniper-VMX-bad-cpu-usage-using-lite-mode-in-kvm-compared-to/td-p/318927)
[https://www.juniper.net/documentation/en_US/vmx/topics/task/configuration/vmx-chassis-flow-caching-enabling.html](https://www.juniper.net/documentation/en_US/vmx/topics/task/configuration/vmx-chassis-flow-caching-enabling.html)

### SQUASHFS issue on vFP

SQUASHFS is a compressed read-only filesystem that is generally used for CD images, liveCD... Getting those error messages may mean that the drive or the media has an issue.
Check you img file (hash fingerprint), and if there is no issue on that side, reboot your VM.

```
SQUASHFS error: squashfs_read_data failed to read block 0x7ca248
SQUASHFS error: Unable to read fragment cache entry [7ca248]
SQUASHFS error: Unable to read page, block 7ca248, size 980f
SQUASHFS error: Unable to read fragment cache entry [7ca248]
SQUASHFS error: Unable to read page, block 7ca248, size 980f
SQUASHFS error: Unable to read fragment cache entry [7ca248]
SQUASHFS error: Unable to read page, block 7ca248, size 980f
SQUASHFS error: Unable to read fragment cache entry [7ca248]
SQUASHFS error: Unable to read page, block 7ca248, size 980f
SQUASHFS error: Unable to read fragment cache entry [7ca248]
SQUASHFS error: Unable to read page, block 7ca248, size 980f
```

### Default passwords

I put the default password for reference here: 
- vCP: root / no password
- vFP: root / root

### Multiple linecards
If you are crazy enough to decide to emulate multiple linecards, here's an interesting link: [https://jncie.eu/how-to-deploy-vmx-with-multiple-res-and-multiple-fpcs-in-eve-ng-kvm/](https://jncie.eu/how-to-deploy-vmx-with-multiple-res-and-multiple-fpcs-in-eve-ng-kvm/)

In that particular case, you have multiple vPFE for one single vCP.

### Mounting issue on vPFE for /dev/sda2

If you got the following error: ```mount: mounting /dev/sda2 on /mnt failed: No such file or directory``` you can correct your virtual hard disk drive type to IDE for this machine.

### Moving VMs and files

In case you want to change the host to increase performance or just to move the VM to another lab environment, I noted that it is generally preferable to have a fresh install instead.
It will spare you some useless troubleshooting time :-)
