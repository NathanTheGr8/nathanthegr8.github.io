---
layout: article
title:  "Getting Started With Wireguard"
key: post20201128
tags: [vpn,server,wireguard]
---
{:refdef: style="text-align: center;"}
![Wireguard Logo](/assets/images/wireguard/wireguard_logo.png){:.rounded}
{: refdef}

Setting a basic wireguard VPN server.

<!--more-->
### Goal

To setup a wireguard vpn with multiple gateways so that if one of them is offline the vpn still works. 
[wireguard IP roaming](https://www.wireguard.com/#built-in-roaming)

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

`cat privatekey` to get the generated private key. We will need to put this into the wireguard config file. Copy and paste the info below into the config. Put your private key and change the CIDR if you want. The PostUP and PostDown are for NAT forwarding if you want to route all your traffic through wireguard. Change eth0 to your server's public ip interface if it is something different.

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
### Clients Install

Wireguard is currently only stable for Linux, but is available for iOS, Android, Windows, MacOS, and more. Go to the [wireguard install page](https://www.wireguard.com/install/) and follow the instructions to download the client for your os. Below are instructions for a second ubuntu 20.04 client. If your client is something else the conf will be the same, but the other steps will differ.

```shell
apt-get update && apt-get upgrade -y
```

Install wireguard 
```shell
apt-get install wireguard
```

Generate your public and private keys.

```shell
cd /etc/wireguard
umask 077; wg genkey | tee privatekey | wg pubkey > publickey
```

`cat privatekey` and `cat publickey` to get the generated keys, we will need them for the wireguard configs. Open the client wireguard config.

```shell
vim /etc/wireguard/wg0.conf
```

```conf
[Interface]
PrivateKey = <Client Private Key>
Address = 192.168.100.2/24

[Peer]
PublicKey = <Server Public Key>
AllowedIPs = 192.168.100.1/24
Endpoint = PublicIP:51820
```

Note I have configured AllowedIPs to only tunnel ips on the `192.168.100.1/24` subnet. If you want to tunnel all of your traffic on the client through wireguard use `0.0.0.0/0`.

Now connect to your server again and add your client information under the `[Peer]` section.

```conf
[Interface]
PrivateKey = <Your Private Key>
Address = 192.168.100.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <Client Public Key>
AllowedIPs = 192.168.100.2
PersistentKeepalive = 25
```

The `PersistentKeepalive = 25` is for clients behind a firewall/NAT (most workstations and mobile phones). If your client has a static public IP, use the `Endpoint` parameter with the clients public IP.

Test the client config with
```shell
wg-quick up wg0
wg-quick down
```

Enable the systemd service and start wireguard on the client.
```shell
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```


### Extras