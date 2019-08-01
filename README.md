# RASPBERRY PI 4 as WireGuard VPN server

Goal: Setup the RASPBERRY PI 4 as VPN server. Clients can establish secure VPN tunnel to the VPN server. Clients will have then internet access through the VPN tunnel.

Setup:

- Hardware: RASPBERRY PI 4 4G Model B (Cortex-A72)
- OS: Raspbian Buster Lite (2019-07-10)

## Install WireGuard

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install raspberrypi-kernel-headers
sudo apt-get install libmnl-dev libelf-dev build-essential pkg-config
sudo reboot

wget https://git.zx2c4.com/WireGuard/snapshot/WireGuard-0.0.20190702.tar.xz
tar xf WireGuard-0.0.20190702.tar.xz
cd WireGuard-0.0.20190702/src
make
sudo make install
```

```bash
# Enable IP Forwarding
# /etc/sysctl.conf set: net.ipv4.ip_forward = 1
sudo perl -pi -e 's/#{1,}?net.ipv4.ip_forward ?= ?(0|1)/net.ipv4.ip_forward = 1/g' /etc/sysctl.conf
sudo reboot

# Validate config change. Expected result: net.ipv4.ip_forward = 1
sysctl net.ipv4.ip_forward
```

## Generate private and public keys for server and client1

```bash
mkdir wgkeys
cd wgkeys

# Server Key-pair
wg genkey > server_private.key
wg pubkey > server_public.key < server_private.key

# Client1 Key-pair
wg genkey > client1_private.key
wg pubkey > client1_public.key < client1_private.key
```

## VPN Server Configuration

```bash
sudo nano /etc/wireguard/wg0.conf

# Edit file content
[Interface]
Address = 10.200.200.1/24
SaveConfig = true
ListenPort = 51820
PrivateKey = <server private key>

# note - substitute eth0 in the following lines to match the Internet-facing interface
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# client1
PublicKey = <Client public key>
AllowedIPs = 10.200.200.2/32
```

```bash
# Start
sudo wg-quick up wg0
# Validating
sudo wg

# Automatically start at startup
sudo systemctl enable wg-quick@wg0
```

## VPN Client Configuration

For the most operation systems the installation of WireGuard is stright forward: https://www.wireguard.com/install/

```bash
sudo nano /etc/wireguard/wg0-client.conf

# File content
[Interface]
Address = 10.200.200.2/24
PrivateKey = <client private key>
# DNS = 10.200.200.1

[Peer]
PublicKey = <server public key>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <VPN server IP>:51820
```

```bash
# Connect to the VPN server
sudo wg-quick up wg0-client
```

Hint: Androids `WireGuard` app can import a wireguard configuration over QR code.

```bash
qrencode -t ASCIIi -r wg0-client.conf
```

## Inspiration

Inspired by:

- https://www.ckn.io/blog/2017/11/14/wireguard-vpn-typical-setup/
- https://github.com/adrianmihalko/raspberrypiwireguard
- https://www.wireguard.com/install/
