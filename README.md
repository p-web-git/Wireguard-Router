# Wireguard And Tailscale Router 
The goal of this page is to document how to configure a router on a linux device so that I can have my own AP (access point) when travelling and be able connect home. 
I tried to run raspap on a docker container and while it is a great solution, I didn't see a good way to integrate with tailscale or a reliable way to always have the AP working

## Hardware

The following configuration will run on Radxa Rock 5c which has 2 wireless interfaces and 1 ethernet port.
- wlan0 will be configured to AP so that we are able to SSH when there in no internet
- wlan1 will be configured to connect to another WIFI with interenet access
- eth0 unsued, but could be bridged to any other interface - TODO

## Access Point

<details>
<summary>Configuration</summary>

### Change the interfaces name - Optional

/etc/systemd/network/10-fixed-eth0-name.link
``` bash
[Match]
MACAddress=d6:56:fd:XX:XX:XX

[Link]
Name=eth0
```
/etc/systemd/network/10-fixed-wlan0-name.link
``` bash
[Match]
MACAddress=88:00:03:XX:XX:XX

[Link]
Name=wlan0
```

/etc/systemd/network/10-fixed-wlan1-name.link
``` bash
[Match]
MACAddress=7c:f1:7e:XX:XX:XX

[Link]
Name=wlan1
```

### Configure static IP address and DHCP server
/etc/systemd/network/20-wlan0-dhcp.network   
``` bash
[Match]
Name=wlan0

[Network]
DHCP=yes
Address=192.168.100.1/24
DHCPServer=yes
IPv6AcceptRA=no
Address=2001:db8::1/64
IPv6SendRA=yes
NTP=pool.ntp.org

[DHCPServer]
EmitDNS=yes
EmitNTP=yes
EmitRouter=yes
PoolOffset=2
PoolSize=99
MaxLeaseTimeSec=864000
```

### DNS

/etc/systemd/resolved.conf 
```bash
DNS=1.1.1.1
DNS=2606:4700:4700::1111
DNSStubListener=yes
```

### Hostapd
``` bash
sudo systemctl enable hostapd.service
sudo systemctl edit --full hostapd.service
```
And add the following line to block any access to the internet that is not through a tunnel - optional:
``` bash
[Service]
...
# This block drops all the traffic from wlan0 to wlan1
ExecStartPre=/usr/sbin/iptables -A FORWARD -i wlan0 -o wlan1 -j DROP
``` 

``` bash
sudo systemctl start hostapd.service
```

 /etc/hostapd/hostapd.conf 
 ``` bash
interface=wlan0
#bridge=br0
driver=nl80211

ssid=SomeSSID
hw_mode=a
channel=149
ieee80211d=1
ieee80211h=1

country_code=GB
ieee80211n=1
ieee80211ac=1

wmm_enabled=1
ht_capab=[HT40+]
vht_oper_chwidth=1
vht_oper_centr_freq_seg0_idx=155

auth_algs=1
wpa=3
wpa_passphrase=SomePassword
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

</details>

## Tunnel

Choose one of the 3 options:
<details>
<summary>Tailscale configuration</summary>

```bash
sudo systemctl enable tailscaled.service
sudo systemctl edit --full tailscaled.service
```
```bash
[Service]
...
ExecStartPost=/sbin/iptables -t mangle -A FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1200
```
⚠️ The above line is required to reduce the maximum TCP packet size to 1200. Whitout this taiscale will drop the packets silently as it as an mtu of 1280.

forward-rules.sh
```bash
#!/bin/bash
WAN_INTERFACE=tailscale0
LAN_INTERFACE=wlan0

eval `iptables --list-rules | grep -P "$LAN_INTERFACE(?!.*DROP)" | sed "s/^-A /iptables -D /g;s/$/;/g"`

iptables -t nat -C POSTROUTING -o $WAN_INTERFACE -j MASQUERADE || iptables -t nat -A POSTROUTING -o $WAN_INTERFACE -j MASQUERADE
iptables -C FORWARD -i $WAN_INTERFACE -o $LAN_INTERFACE -m state --state RELATED,ESTABLISHED -j ACCEPT || iptables -A FORWARD -i $WAN_INTERFACE -o $LAN_INTERFACE -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -C FORWARD -i $LAN_INTERFACE -o $WAN_INTERFACE -j ACCEPT || iptables -A FORWARD -i $LAN_INTERFACE -o $WAN_INTERFACE -j ACCEPT

# Rule already set by /etc/systemd/system/tailscaled.service'
#iptables -t mangle -C FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1200 || iptables -t mangle -A FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1200
iptables-save
```

Start with:
```bash
sudo systemctl start tailscaled.service
sudo tailscale status    # copy the ip from the exit node
sudo tailscale up --accept-dns --accept-routes --exit-node=100.XX.XX.XX --exit-node-allow-lan-access
sudo chmow +x forward-rules.sh
sudo ./forward-rules.sh
```
</details>

<details>
<summary>Wireguard configuration</summary>

/etc/wireguard/wg0.conf                                                                                            
```bash
[Interface]
PrivateKey = ...

# Enable the follwing rules if we only want Wlan0 to use wireguard interface 
Table = 200
PostUp = ip rule add iif wlan0 table 200
PostDown = ip rule del iif wlan0 table 200

# Delete any rules that are forwarding routing directly to wlan1
PreUp = eval `iptables --list-rules | grep 'wlan1.*-j ACCEPT' | sed 's/^-A /iptables -D /g;s/$/;/g'`

# Rules for VPN routing and NAT 
PreUp = iptables -t nat -A POSTROUTING -o %i -j MASQUERADE
PreUp = iptables -A FORWARD -i %i -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
PreUp = iptables -A FORWARD -i wlan0 -o %i -j ACCEPT

# https://docs.pi-hole.net/guides/vpn/wireguard/internal/#enable-nat-on-the-server
PostUp = iptables -w -t nat -A POSTROUTING -o wlan1 -j MASQUERADE
PostDown = iptables -w -t nat -D POSTROUTING -o wlan1 -j MASQUERADE

# At the end, remove Rules of VPN and NAT
PostDown = iptables -t nat -D POSTROUTING -o %i -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -D FORWARD -i wlan0 -o %i -j ACCEPT

# This block drops all the traffic from wlan0 to wlan1
PostDown = iptables -C FORWARD -i wlan0 -o wlan1 -j DROP || iptables -A FORWARD -i wlan0 -o wlan1 -j DROP

[Peer]
PublicKey = ...
```
⚠️ I lost the ability to use ipv6 on the router so I add the rules `table 200` so that only the clients connected to wlan0 use wireguard. 
<details>
<summary>Alternativly to set Table 200, drop the preference of using ipv6</summary>
 
/etc/gai.conf
```bash
precedence ::ffff:0:0/96  100
```
</details>

Start with:
```bash
sudo wg-quick up wg0
```

</details>


<details>
<summary>No tunel configuration</summary>

forward-rules.sh
```bash
#!/bin/bash
WAN_INTERFACE=wlan1
LAN_INTERFACE=wlan0

eval `iptables --list-rules | grep -P "$LAN_INTERFACE" | sed "s/^-A /iptables -D /g;s/$/;/g"`

iptables -t nat -C POSTROUTING -o $WAN_INTERFACE -j MASQUERADE || iptables -t nat -A POSTROUTING -o $WAN_INTERFACE -j MASQUERADE
iptables -C FORWARD -i $WAN_INTERFACE -o $LAN_INTERFACE -m state --state RELATED,ESTABLISHED -j ACCEPT || iptables -A FORWARD -i $WAN_INTERFACE -o $LAN_INTERFACE -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -C FORWARD -i $LAN_INTERFACE -o $WAN_INTERFACE -j ACCEPT || iptables -A FORWARD -i $LAN_INTERFACE -o $WAN_INTERFACE -j ACCEPT

# Rule already set by /etc/systemd/system/tailscaled.service'
#iptables -t mangle -C FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1200 || iptables -t mangle -A FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1200
iptables-save
```
Start with:
```bash
sudo chmow +x forward-rules.sh
sudo ./forward-rules.sh
```
</details>
