---
layout: article
title:  "Getting Started With Wireguard"
key: post20201128
tags: [vpn,server,wireguard]
---
{:refdef: style="text-align: center;"}
![Wireguard Logo](/assets/images/wireguard/wireguard_logo.png){:.rounded}
{: refdef}

Setting a basic wireguard VPN server. See how to configure the server, desktop, and mobile peers.

<!--more-->

### Server Install
Instructions are for a fresh Ubuntu 20.04, but should work for most debian based distros.

First update and upgrade your current server.
```shell
sudo apt-get update && apt-get upgrade -y
```


Enable IP forwarding by uncommenting `net.ipv4.ip_forward=1` in sysctl.conf.
```shell
sudo vim /etc/sysctl.conf
```

`sudo sysctl -p` or reboot to load the changes

Setup firewall some kind of firewall, I typically use [UFW](https://wiki.archlinux.org/index.php/Uncomplicated_Firewall). I will allow ssh, port 22, and 51820, for wireguard.

```shell
sudo ufw allow ssh
sudo ufw allow 51820/udp
sudo ufw enable
```

Finally we can install wireguard. Note some older distros will use [dkms](https://wiki.archlinux.org/index.php/Dynamic_Kernel_Module_Support) if they have a kernel without wireguard built in (Linux 5.6 or lower). DKMS usally works great but can occasionally fail to load after an update or reboot. I tend to avoid it because of experiences with zfs's dkms modules[20.04 doesn't need dkms](https://www.phoronix.com/scan.php?page=news_item&px=Ubuntu-20.04-Adds-WireGuard) but uses a 5.4 kernel with wireguard compiled in. 

```shell
sudo apt-get install wireguard
```

Change to the new wireguard directory and generate a public and private key.

```shell
cd /etc/wireguard
sudo umask 077; wg genkey | tee privatekey | wg pubkey > publickey
```

`cat privatekey && cat publickey` to get the generated keys. We will need to put this into the wireguard config file. Copy and paste the info below into the config. Put your private key and change the CIDR if you want. The PostUP and PostDown are for NAT forwarding if you want to route all your traffic through wireguard. Change eth0 to your server's public ip network interface if it is something different. Use `ip a` or `ifconfig` if you are not sure what your interfaces are.

```shell
sudo vim /etc/wireguard/wg0.conf
```

```conf
[Interface]
PrivateKey = <Server Private Key>
Address = 192.168.100.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```


Test the config with
```shell
sudo wg-quick up wg0
sudo wg-quick down
```

Enable the systemd service and start wireguard
```shell
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

You can also get information about the running wireguard service by using the `wg` command.
```shell
nathan@davis-wiregurard:/etc/wireguard# sudo wg
interface: wg0
  public key: [...]
  private key: (hidden)
  listening port: 51820

peer: [peer public key]
  endpoint: [Public IP]:9846
  allowed ips: 192.168.100.4/32
  latest handshake: 47 seconds ago
  transfer: 61.95 KiB received, 204.93 KiB sent
  persistent keepalive: every 25 seconds

peer: [peer public key]
  endpoint: [Public IP]:56853
  allowed ips: 192.168.100.7/32
  latest handshake: 1 minute, 14 seconds ago
  transfer: 410.17 KiB received, 140.44 KiB sent
  persistent keepalive: every 25 seconds

  ...
```
### Clients Install

Wireguard is currently only stable for Linux, but is available for iOS, Android, Windows, MacOS, and more. Go to the [wireguard install page](https://www.wireguard.com/install/) and follow the instructions to download the client for your os. Below are instructions for a second ubuntu 20.04 client. If your client is something else the conf will be the same, but the other steps will differ.

```shell
sudo apt-get update && apt-get upgrade -y
```

Install wireguard 
```shell
sudo apt-get install wireguard
```

Generate your public and private keys.

```shell
sudo cd /etc/wireguard
sudo umask 077; wg genkey | tee privatekey | wg pubkey > publickey
```

`cat privatekey` and `cat publickey` to get the generated keys, we will need them for the wireguard configs. Open the client wireguard config.

```shell
sudo vim /etc/wireguard/wg0.conf
```

```conf
[Interface]
PrivateKey = <Client Private Key>
Address = 192.168.100.2/32

[Peer]
PublicKey = <Server Public Key>
AllowedIPs = 192.168.100.1/24
Endpoint = PublicIP:51820
```

Note I have configured AllowedIPs to only tunnel IPv4s on the `192.168.100.1/24` subnet. If you want to tunnel all of your traffic on the client through wireguard use `0.0.0.0/0, ::/0`.

Now connect to your server again and add your client information under the `[Peer]` section.

```conf
[Interface]
[...]

[Peer]
PublicKey = <Client Public Key>
AllowedIPs = 192.168.100.2/32
PersistentKeepalive = 25
```

The `PersistentKeepalive = 25` is for clients behind a firewall/NAT (most workstations and mobile phones). If your client has a static public IP, use the `Endpoint` parameter with the clients public IP.

Test the client config with
```shell
sudo wg-quick up wg0
sudo wg-quick down
```

Enable the systemd service and start wireguard on the client.
```shell
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```
You may want to begin adding additional clients to your wireguard vpn. Just add another peer section for each client in the server config. The Client config will only ever need to contain the public gateway server as a peer. 

### Mobile devices

Adding mobile devices can be a pain because it can be a hassle to securely exchange the wireguard public and private keys. I would recommend using [qrencode](https://github.com/fukuchi/libqrencode) to make a scannable qr code that can be imported on the iOS and android wireguard clients. On debian systems you can install the app with `sudo apt install qrencode`

Generate your mobile devices keys on your server. Note I am naming them `mobile-`, but the name can be anything you like.

```shell
cd /etc/wireguard
wg genkey | tee mobile-privatekey | wg pubkey > mobile-publickey
```
Make the mobile config file
```shell
cd /etc/wireguard
sudo vim mobile.conf

[Interface]
PrivateKey = <contents of mobile-privatekey>
Address = 192.168.100.x/32 #replace with your desired IPv4 address

[Peer]
PublicKey = <contents of server-publickey>
Endpoint = <server-ip>:51820
AllowedIPs = 192.168.100.0/24
```

Generate the QR code
```shell
qrencode -t ansiutf8 < /etc/wireguard/mobile.conf
```

{:refdef: style="text-align: center;"}
![Mobile QR Code](/assets/images/wireguard/mobile-QR-code.png){:.rounded}
{: refdef}

Simply open the mobile wireguard app, hit the + button, and click scan from QR code.

### Extras

For additional help I would recommend checking out the [Arch Wiki](https://wiki.archlinux.org/index.php/WireGuard). Even if you don't use arch linux, it is a valuable resource. 

Also starting in Android 12 Wireguard may be [native to the 4.19 and 5.4 kernel](https://www.xda-developers.com/google-adds-wireguard-vpn-android-12-linux-kernel-5-4/). 

[wireguard IP roaming](https://www.wireguard.com/#built-in-roaming)