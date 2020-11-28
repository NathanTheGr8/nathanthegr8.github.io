---
layout: article
title:  "HA Wireguard"
key: post20201128
tags: [vpn,server,wireguard]
---
{:refdef: style="text-align: center;"}
![OpenZFS Logo](/assets/images/wireguard/wireguard_logo.png){:.rounded}
{: refdef}

Setup Wireguard

<!--more-->
### Server Install
Instructions are for a fresh Ubuntu 20.04, but should work for most debian based distros.

First update and upgrade your current server.
```shell
apt-get update && apt-get upgrade -y
```


Enable IP forwarding by uncommenting `net.ipv4.ip_forward=1` in sysctl.conf.
```shell
vim /etc/sysctl.conf
```

`sysctl -p` or reboot to load the changes

Setup firewall some kind of firewall, I typically use [UFW](https://wiki.archlinux.org/index.php/Uncomplicated_Firewall). I will allow ssh, port 22, and 51820, for wireguard.

```shell
ufw allow ssh
ufw allow 51820/udp
ufw enable
```

Finally we can install wireguard. Note some old distros will use [dkms](https://wiki.archlinux.org/index.php/Dynamic_Kernel_Module_Support) if they have a kernal without wireguard built in. [20.04 doesn't need dkms](https://www.phoronix.com/scan.php?page=news_item&px=Ubuntu-20.04-Adds-WireGuard) but uses a 5.4 kernal with wireguard compiled in.
```shell
apt-get install wireguard
```

Change to the new wireguard directory and generate a public and private key.

```shell
cd /etc/wireguard
umask 077; wg genkey | tee privatekey | wg pubkey > publickey
```

`cat privatekey` to get the generated private key. We will need to put this into the wireguard config file.

```shell
vim /etc/wireguard/wg0.conf
```

```conf
[Interface]
PrivateKey = <Your Private Key>
Address = 192.168.100.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```


Test the config with
```shell
wg-quick up wg0
wg-quick down
```

Enable the systemd service and start wireguard
```shell
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```
### Second Server


```shell
apt-get update && apt-get upgrade -y
```

```shell
vim /etc/sysctl.conf
```
uncomment `net.ipv4.ip_forward=1`

`sysctl -p` or reboot to load the changes

Install wireguard 
```shell
apt-get install wireguard
```


```shell
cd /etc/wireguard
umask 077; wg genkey | tee privatekey | wg pubkey > publickey
```

`cat privatekey` to get the generated private key

```shell
vim /etc/wireguard/wg0.conf
```

```conf
[Interface]
PrivateKey = <Your Private Key>
Address = 192.168.100.2/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <Server Public Key>
AllowedIPs = 192.168.100.1/24
#Endpoint = 178.128.150.185:51820
Endpoint = PublicIP:51820



[Peer]
# Nathan Windows Desktop
PublicKey = 3/Vxl/Q94glOhy/IuthZqFFmhtId6MvMnUGDw0H6IlA=
AllowedIPs = 192.168.100.7
PersistentKeepalive = 25
```


Test the config with
```shell
wg-quick up wg0
wg-quick down
```

Enable the systemd service and start wireguard
```shell
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```


### Client Install