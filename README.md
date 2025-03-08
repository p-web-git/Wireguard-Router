# Wireguard And Tailscale Router 
The goal of this page is to document how to configure a router on a linux device so that I can have my own AP (access point) when travelling and be able connect home. 

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

