# Raspberry Pi

The following is a step-by-step guide on how to setup a brand-new Raspberry Pi 4 as a wireless
hotspot with internet access through a remote VPN server. The overall setup looks something like
this:

![image](./assets/diagram.png)

As a result, any device connected to our Raspberry Pi gains indirect access to the Internet via the
VPN tunnel. If the connection to the VPN server is lost, the tunnel closes, preventing packets from
reaching the Internet.

## Setup Guide

This document assumes that the network device used as a wireless access point is named `ap0` and the
network device connected to the internet as `inet0`.

In order to set appropriate adapter names in your system, add the following to
`/etc/udev/rules.d/70-netnaming.rules`:

```
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="<ap0   mac address>", NAME="ap0"
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="<inet0 mac address>", NAME="inet0"
```

Additionally, make sure that the access point device is not used to connect to the internet since we
do not want to manage two separate networks by a single interface. We can do that by adding the
following to `/etc/NetworkManager/conf.d/unmanaged.conf`:

```
[keyfile]
unmanaged-devices=interface-name:ap0
```

After that, restart `NetworkManager` and connect to the internet using the appropriate network
device (`inet0`).

### Prerequisites

Before following the instructions below, we must ensure our system is up-to-date and has all the
expected packages required for this setup.

At first, the fresh Raspberry Pi OS may require an update:

```sh
sudo apt-get update && sudo apt-get upgrade -y
```

Then, run the following command to ensure all of the tools required for this setup are available:

```sh
sudo apt-get install vim ufw hostapd dnsmasq dhcpcd
```

### Setup the Wi-Fi link layer.

This step is necessary in order to enable wireless clients to associate to our access point and
exchange IP packets with it.

#### IP forwarding

The first thing we need to do is enable packet forwarding (i.e. allow traffic to flow from one
network to another). This can be done by adding the following to `/etc/sysctl.d/30-ipforward.conf`:

```
net.ipv4.ip_forward = 1
# net.ipv4.conf.all.forwarding = 1
# net.ipv6.conf.all.forwarding = 1
```

#### Disable IPv6

To further simplify a few things ahead of time, we can disable IPv6 support by modifying
`/etc/sysctl.d/40-ipv6.conf`:

```
net.ipv6.conf.all.disable_ipv6 = 1
# net.ipv6.conf.nic0.disable_ipv6 = 1
# ...
# net.ipv6.conf.nicN.disable_ipv6 = 1
```

At this point, restart the `systemd-sysctl` service to apply the changes.

#### Assign static IP to `ap0`

Append the lines below to the end of `/etc/dhcpcd.conf`:

```
interface ap0
    static ip_address=5.9.0.1
    nohook
```

You can change `5.9.0.1` to any other address as long as it does not conflict with existing
addresses and subnets on your system.

You may need to restart `dhcpcd.service` after that.

#### Configure DHCP (+DNS)

At this point, we need a way to assign IP addresses to new hotspot clients. For that, we have
installed `dnsmasq` earlier. It will additionally handle incoming DNS requests for us.

The configuration file at `/etc/dnsmasq.conf` should look something like this:

```
interface=ap0
bind-dynamic

listen-address=127.0.0.1

dhcp-option=3,5.9.0.1

dhcp-range=5.9.0.10,5.9.0.100,24h
```

As always, don't forget to restart `dnsmasq.service`.

#### Setup hostapd

Lastly, we must configure the `hostapd` service that will turn our network device into the desired
access point.

This is what I have in `/etc/hostapd/hostapd.conf`:

```
interface=ap0

driver=nl80211
hw_mode=a
channel=0
ieee80211d=1
country_code=DE
ieee80211n=1
ieee80211ac=1
ieee80211d=0
ieee80211h=0
wmm_enabled=1
require_nt=1
ht_capab=[MAX-AMSDU-3839][HT40+][SHORT-GI-20][SHORT-GI-40][DSSS_CCK-40]
require_vht=1
vht_capab=[MAX-AMSDU-3839][SHORT-GI-80]
vht_oper_chwidth=1

ssid=raspi

auth_algs=1
wpa=2
wpa_pairwise=CCMP
wpa_key_mgmt=WPA-PSK
wpa_passphrase=********
```

Keep in mind that certain fields may depend on the network device in use. See
[docs](https://wireless.wiki.kernel.org/en/users/Documentation/hostapd) for more information.

### Network configuration

At this stage, the AP should function like any other router, except that it currently does not
provide internet access. But we'll address that in a bit.

#### Firewall

The AP needs a way to manage incoming and outgoing IP packets which can be accomplished with a
firewall. If you haven’t enabled it yet, you may need to do so.

```sh
sudo systemctl enable --now ufw && sudo ufw enable
```

That's it. Now, let's block any incoming or outgoing traffic.

```sh
sudo ufw default deny incoming && sudo ufw default deny outgoing
```

The AP should stop working at this point. Let's fix that by adding a few exception rules.

```sh
sudo ufw allow in from any port 68 to any port 67 proto udp comment "Allow incoming DHCP traffic"
sudo ufw allow in to any port 53 comment "Allow incoming DNS traffic"
sudo ufw allow in to any port 22 comment "Allow incoming SSH traffic"

sudo ufw allow out to 5.9.0.0/24 comment "Allow outgoing LAN traffic"
```

These should allow AP-related packets to pass through.

#### VPN

At this point, it's advisable to refer to your VPN service documentation. Typically, you just need
to download an `.ovpn` file from your VPN provider and import it into `NetworkManager`:

```sh
nmcli con import type openvpn file <ovpn-file>
```

Following this, you should add additional exception rules to `ufw` to allow `OpenVPN` to reach the
destination VPN server.

Once the connection to the server is established, `OpenVPN` will create a VPN tunnel (typically
`tun0`). To enable internet access through this tunnel, we need to add an additional rule to `ufw`:

```sh
sudo ufw allow out on tun0 to any from any
```

Now, outgoing packets will only pass through this tunnel. If the connection to the VPN server is
lost, the tunnel closes, preventing packets from reaching the Internet.

#### Internet access

In the final step, we just need to enable NAT on `tun0` and allow packets from `ap0` to reach it.

To do that, add the following to `/etc/ufw/before.rules` just before the filter rules.

```
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -o tun0 -j MASQUERADE
COMMIT
```

And finally, append this line below right after the filter rules:

```
-A FORWARD -i ap0 -o tun0 -j ACCEPT
```

Now, AP clients should have internet access through a remote VPN server.
