---
layout: article
title:  "ZFS Pool Expansion"
key: post20201123
tags: [zfs,server]
---
{:refdef: style="text-align: center;"}
![OpenZFS Logo](/assets/images/zfs-pool-expansion/open_zfs.png){:.rounded}
{: refdef}

My zfs pool was has been getting close to full (%70%) so I started exploring my options. I found a good deal on 16TB hdds so I decided to buy 6 of them and expand my pool. In this post I explore my options for expanding my zfs pool.

<!--more-->
### The Options

Earlier this month I bought 6 [ST16000NM001G](https://www.seagate.com/www-content/datasheets/pdfs/exos-x16-DS2011-3-2008US-en_US.pdf) drives. I debated between 3 options for expanding my storage.

1.  Create a new raidz2 vdev and add it to my existing pool
    
    This is what I have done in the past to expand my pool. However I currently have 4 6 disk vdevs (24 drives) plus 2 ssds. My current server chassis can hold 36 3.5 in hdds. If I add 6 hdds in a new vdev I won't be able to expand the array in the same chassis. If I did want to expand the array I would do so with the command below.

    ```bash
    zpool add storage raidz2 
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_1
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_2
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_3
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_4
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_5
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_6
    ```

2. Create a new pool with a raidz2 vdev and migrate my data off the old pool.

    The would give me a new pool with ~60 TB of usable storage. My current pool has about 90TB of usable storage with about 65TB used, so I couldn't move everything to the new pool. So this option would still use 30 disks. The command to create the new pool is below.

    ```bash
    zpool create storage2 raidz2 
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_1
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_2
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_3
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_4
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_5
    /dev/disk/by-id/ata-ST16000NM001G-2KK103_6
    ```

3. Replace all 6 of the smaller drives in an existing vdev expanding it my new larger drives

    I would pick the oldest vdev and replace every 6TB drive with a new 16TB drive. This option would give me about 40 TB of additional usable space. I would then test the old 6 TB drives and use them as hot and cold spares for the rest of the pool. This is the option I went with since it leaves me with future expansion options.

    ```shell
    zpool replace storage /dev/disk/by-id/ata-ST6000VN0033-2EE110_1 /dev/disk/by-id/ata-ST16000NM001G-2KK103_1

    zpool replace storage /dev/disk/by-id/ata-HGST_HDN726060ALE610_2 /dev/disk/by-id/ata-ST16000NM001G-2KK103_2

    zpool replace storage /dev/disk/by-id/ata-HGST_HDN726060ALE610_3 /dev/disk/by-id/ata-ST16000NM001G-2KK103_3

    zpool resilver storage

    ## wait for resilver to finish

    zpool replace storage /dev/disk/by-id/ata-HGST_HDN726060ALE610_4 /dev/disk/by-id/ata-ST16000NM001G-2KK103_4

    zpool replace storage /dev/disk/by-id/ata-HGST_HDN726060ALE610_5 /dev/disk/by-id/ata-ST16000NM001G-2KK103_5

    zpool replace storage /dev/disk/by-id/ata-HGST_HDN726060ALE614_6 /dev/disk/by-id/ata-ST16000NM001G-2KK103_6

    zpool resilver storage
    ```

### ZFS Auto Expand

I left the old drives in the chassis and replaced 3 drives at a time. I run the [zpool resilver](https://openzfs.github.io/openzfs-docs/man/8/zpool-resilver.8.html) command so that the 3 resilver operations will happen in parallel. Otherwise the second 2 replace operations will wait for the 1st to finish. There is no additional risk in doing [multiple zpool replace](https://github.com/openzfs/zfs/pull/7732) operations at the same time, since the original disks are still plugged in and healthy. 

> There is no additional risk in doing multiple zfs replace operations at the same time, since the original disks are still plugged in and healthy. 

After replacing my disks I was surprised to see `zfs list` didn't show any additional space. This is because I forgot to set the auto expand property on my pool. 

```shell
zpool set autoexpand=on storage
```

Since I forgot to set that property I had to go through and [online -e](https://openzfs.github.io/openzfs-docs/man/8/zpool-online.8.html) each hard drive.

```bash
zpool online -e storage /dev/disk/by-id/ata-ST16000NM001G-2KK103_1
zpool online -e storage /dev/disk/by-id/ata-ST16000NM001G-2KK103_2 
zpool online -e storage /dev/disk/by-id/ata-ST16000NM001G-2KK103_3 
zpool online -e storage /dev/disk/by-id/ata-ST16000NM001G-2KK103_4 
zpool online -e storage /dev/disk/by-id/ata-ST16000NM001G-2KK103_5
zpool online -e storage /dev/disk/by-id/ata-ST16000NM001G-2KK103_6
```

After expanding each drive I now had the additional space I was expecting.

```shell
nathan@pve:~# sudo zfs list storage
NAME      USED  AVAIL     REFER  MOUNTPOINT
storage  72.5T  53.8T     64.8T  /storage
nathan@pve:~# 
```
                                                                                        
