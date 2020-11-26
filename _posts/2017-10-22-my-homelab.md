---
layout: article
title:  "My Homelab"
key: post20171022
tags: [homelab, server]
---

> Welcome, all. This is my first post about my home lab setup.

<!--more-->
{:refdef: style="text-align: center;"}
![Picture of my server](/assets/images/my_homelab/homelab.jpg){:.rounded}
{: refdef}

### Introduction

My homelab consists of a 4U Supermicro server with the following specs

* Motherboard – X9DRH-iF
* Case – 4 U SC847E16-R1K28LPB
* CPU – Dual Xeon E5-2680 v2 2.80GHz 10-CORE 115TDP
* 18 6TB HDD of various manufacturers.
* A 128 GB sataDom boot drive
* 128 GB of ECC ram

I also have a used APC UPS, a Protectli pfsense firewall, a few unmanaged switches, and a cluster of 5 raspberry pis.

I have bought most if not all of the equipment used. My 4U server’s primary purpose is to run a Plex media server and serve as “Netflix” for my friends and family. The server runs a host of web apps related to the media server, a few public websites, a home security monitor system, a windows test lab, and a few other random things.

### The OS
My server runs the Proxmox Hypervisor. I choose a hypervisor because I did want to run a few fully virtualized clients for testing and learning. I chose this hypervisor over Xen Server, ESXi, Unraid, FreeNAS, or just plain KVM because it was based off Debian Linux, which I am very familiar with, it has an easy to use web interface for managing KVM clients, and the general recommendation of the /r/homelab subreddit. Proxmox has its quarks and lacks some features like live VM migration(edit this was added in v5.0), but it has every feature I need/want. Things like

* Being fully open source
* have  great community support and guides
* Easy to use web UI
* and other reasons

{:refdef: style="text-align: center;"}
![zpool status](/assets/images/my_homelab/zfsstats.png){:.rounded}
{: refdef}

### The  Storage

I have trusted my data to the battle-tested and mature file system ZFS. I chose ZFS over hardware raid, btrfs, SnapRAID, and other because of its

1. Data Integrity
    - checksumming on everything to protect against bitrot and errors in I/O
    copy-on-write for everything (atomic transactions, crash-proof)
    dynamic parity stripe width, so no raid 5 write vulnerability!
2. Ease of Use
    - While ZFS does require you to learn a few CLI commands, they are dead simple and easily googleable.
3. Excellent Documentation
    - ZFS has existed since ~2004 and the ZFS on Linux project since 2008. Any question or idea you have is easy to find on stack overflow or another forum.
    The recommendations of a friend Bibi Zou.

ZFS’s brillance amazes me every day. It is the most elegant and future proof piece of software I have ever used. The only downside (for my use case) to ZFS is that you have to add disks in vdev groups, you can’t just add one disk at a time. My current zfs pool consists of 18 6TB drives. The drives are split into three vdevs of 6 drives in a raidz2 array. This gives me a lot of resiliency against drive failures. Furthermore, the copy on write nature of the file system allows for “free” snapshots. Snapshots only use storage space when the files are changed or deleted.

{:refdef: style="text-align: center;"}
![Portainer Containers](/assets/images/my_homelab/portainerStats.png){:.rounded}
{: refdef}

### Docker and VMs

I try to isolate all the apps and software on my server. This keeps the machine “cleaner” with out random trace files from previous failed experiments. To do this, I rely on Docker containers and virtual machines. 90% of the apps on my server run in Docker containers. Most of the containers are community maintained images from Github, but a few are my own custom images. Since Proxmox doesn’t natively support the management of Docker containers (they use LXC), I use Portainer.

Portainer gives me a  nice web UI to view all my containers. I can graphically see logs for each container, see performance stats, and even start a bash session in each container from the web. A nice feater about Portainer is that it is just a docker container! The only downside to Portainer is that it doesn’t store performance data for your containers, you have to get another service or app for that.

While most my apps run fine in containers, there are a few things I want VMs for, like my windows test lab. My test lab consist of a two windows 2016 servers, a windows ten machine, and a virtualized pfsense router. The lab is on its own VLAN with all traffic routed through the pfsense router. I run domain controller, SCCM server, and plan to make and MDT server. I use it for experimentation, testing, and learning.

I also have a few random Linux distro VMs I use for random testing.

{:refdef: style="text-align: center;"}
![Pfsnese log](/assets/images/my_homelab/pfsenseLogo.png){:.rounded}
{: refdef}

### Firewall and Router
For the longest time, I justed used a consumer access point, switch, router, and firewall combo unit from ASUS. It worked fine for basic port forwarding and a VPN, but I wanted to experiment with intrusion detection systems like snort, so I went searching for a firewall with a more advanced feature set that wouldn’t cost a fortune. I considered used units from Cisco, Sophos, and Ubiquiti but I prefer open source technologies for my homelab. I find the are easier to tinker with, not to mention cheaper. I had heard about Pfsense and experimented with it in VM. I loved it’s easy to use and modern web UI. Plus there is a great comunity around it and a plethora of addon packages. Some of the packages I use are

1. PfBlockerNG:
- Manage IPv4/v6 List Sources to allow/deny based on items such as the geolocation of an IP address, the domain name of a resource, or the Alexa ratings of particular websites. I use it to block access to my server from foreign countries, top spammers, and a few other lists.
2. Snort
- Snort is an open source network intrusion prevention system, capable of performing real-time traffic analysis and packet logging on IP networks. I use snort to help protect my network and websites from attackers. It is very intensive so you can’t just run it on a raspberry pi.
3. OpenVPN-client-export
- This is just an add-in that allows for easy export of VPN configuration settings for clients.
4. ntopng
- Ntopng is a network probe that shows network usage in a way similar to what top does for processes. It gives me a nice over view of the types of traffic going on in my network.
