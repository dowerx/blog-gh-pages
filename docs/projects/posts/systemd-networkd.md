---
date:
  created: 2024-12-06
tags:
  - systemd
  - networking
  - linux
  - vpn
  - wireguard
  - openvpn
comments: true
---

# How to configure systemd-networkd

These are some tipps I learned while setting up networking for my desktop and some VMs.

<!-- more -->

!!! info "Sadly, I don't know about any extensions or manager software for WiFi and *systemd-networkd*, so these only apply to wired connections. You could set up your links using *networkd* but then you would neet to use wpa_supplicant or something similar to connect to any networks."

## 1. Install it
``` sh
apt install systemd-networkd systemd-resolved
systemctl enable --now systemd-networkd systemd-resolved
```
!!! warning "DNS resolution won't work until we configure *networkd*. Make sure you don't need it until then!"

## 2. Disable NetworkManager
By default, desktop installs of Debian use NetworkManager, we won't be needing it. 🫡

``` sh
systemctl disable NetworkManager
systemctl stop NetworkManager
```

## 3. Configure the physical interface

### 3.1. Link
We will change the name of the interface so we can reference it later.

``` systemd
# /etc/systemd/network/10-lan0.link
[Match]
MACAddress=40:b0:76:7b:62:59

[Link]
Name=lan0
WakeOnLan=magic
```

!!! note "WakeOnLan is used to turn on the device remotely with a magic packet if it is in standby and connected to the network."

### 3.2. Bridge network
Later on we will create a bridge interface for running VMs. We need to tell the physical interface which bridge to forward traffic to.

``` systemd
# /etc/systemd/network/20-lan0.network
[Match]
Name=lan0

[Network]
Bridge=br0
```

## 4. Configure a bridge interface

### 4.1. The device
We first define the bridge:

``` systemd
# /etc/systemd/network/30-br.netdev
[NetDev]
Name=br0
Kind=bridge
MACAddress=none
```

!!! note "We set `MACAddress` to `none` so that the interface copies the MAC of it's first slace interface (in this case *lan0*'s)."

### 4.2. Bridge Link
We need this for the MAC configuration.

``` systemd
# /etc/systemd/network/40-br0.link
[Match]
OriginalName=br0

[Link]
MACAddressPolicy=none
```

### 4.3. Bridge network
We can choose to configure a static IP or use DHCP.
For servers I recommend going with static IPs.

#### 4.3.1. Static IP

``` systemd
# /etc/systemd/network/50-br0.network
[Match]
Name=br0

[Network]
Address=192.168.11.128/24
Gateway=192.168.11.1
DNS=1.1.1.3#family.cloudflare-dns.com
DNS=1.0.0.3#family.cloudflare-dns.com
DNSOverTLS=yes
```

!!! note "I configured it to use `DNSOverTLS` so my DNS querys are encrypted. You don't need to do this but it is more secure. If you choose regular DNS leave out the `#family.cloudflare-dns.com` part and set `DNSOverTLS` to `no`"

#### 4.3.2. DHCP
``` systemd
# /etc/systemd/network/50-br0.network
[Match]
Name=br0

[Network]
DHCP=yes
```

#### 4.3.3. Custom static routes
You may need to configure static routes. You can do this by adding a `[Route]` section to your bridge's network configuration.
``` systemd
# /etc/systemd/network/50-br0.network
...
[Route]
Gateway=192.168.11.232
Destination=192.168.50.0/24
GatewayOnLink=yes
...
```

## 5. Wireguard
Wireguard is an opensource VPN protocol running in a kernel module. Here is how you can configure a client peer for it.

### 5.1. Install Wireguard
``` sh
apt install wireguard
```

### 5.2. Place your keys
Place your preshared key in */etc/systemd/network/wg0.key* and your private key in */etc/systemd/network/wg0.privkey*

You need to make sure these keys are not readable by anybody but the root user.
``` sh
chown root:systemd-network /etc/systemd/network/wg0.*
chmod 640 /etc/systemd/network/wg0.*
```

### 5.3. Wireguard NetDev
``` systemd
# /etc/systemd/network/60-wg0.netdev
[NetDev]
Name=wg0
Kind=wireguard

[WireGuard]
PrivateKeyFile=/etc/systemd/network/wg0.privkey

[WireGuardPeer]
PublicKey=eeno6ac4thicaixeeceibee5Phoh9Eetheimapa9boo=
PresharedKeyFile=/etc/systemd/network/wg0.key
Endpoint=example.org:51820
AllowedIPs=192.168.24.0/24
PersistentKeepalive=10
```

### 5.4. Wireguard network
``` systemd
# /etc/systemd/network/60-wg0.netdev
[Match]
Name=wg0

[Network]
Unmanaged=yes
Address=192.168.24.3/24
DNS=192.168.24.1

[Route]
Gateway=192.168.24.1
Destination=192.168.24.0/24
GatewayOnLink=yes
```

## 6. OpenVPN
OpenVPN is another VPN protocol. It sadly can't be configured with *networkd* but can run as a *systemd* service, the second best thing.

### 6.1. Install OpenVPN
``` sh
apt install openvpn
```

### 6.2. Place config
Move your configuration files to */etc/openvpn/client*. Change the extension of the *.ovpn* file to *.conf*.

### 6.3. Start the service
``` sh
systemctl start openvpn-client@${CONFIG_NAME}
```
Replace `${CONFIG_NAME}` with the name of your *.conf* file without the extension.

You can disconnect by stopping it.
``` sh
systemctl start openvpn-client@${CONFIG_NAME}
```

!!! note "You can configure this service to start at boot by enabeling it. `#!sh systemctl enable openvpn-client@${CONFIG_NAME}`"

!!! warning "OpenVPN pulls its routes from the server, even if they conflict with you local network. This means that you may loose connection to your local gateway because OpenVPN tries to route your LAN traffic through the VPN. You can drop a specific route from the pulled list by setting a pull filter in you *.conf* file: `pull-filter ignore "route 192.168.11."`"