# Project goals

Creating a Linux router with firewall, WiFi, VPN, NAS and other services on PC Engines APU4 computer. Device will bridge several ethernet ports and WiFi with one master ethernet port connected to gateway (with DHCP server). MAC whitelisting will be implemented on wireless interface.

# Motivation

To learn more about networking in Linux.

# Setting up APU4D4 and installing OS

## 0. Prerequisities

- [PC Engines APU4D4](https://www.pcengines.ch/apu4d4.htm) with power adapter and chassis
- [Mikrotik R11e-2HnD](https://mikrotik.com/product/R11e-2HnD) or similar miniPCI-E WiFi card
- [mSATA SSD module](https://pcengines.ch/msata30b.htm)
- USB to RS232 convertor
- U.FL to SMA cables
- 2.4 GHz WiFi antenna
- (optional) disk for NAS storage, either SATA or USB

## 1. Setting up PC Engines APU4D4

Insert all desired miniPCI-E / mSATA cards (note that only one slot is mSATA, double check that you placed SSD module there). Connect U.FL connectors to WiFi card. Assemble the cover as per [official instructions](https://www.pcengines.ch/apucool.htm).

## 2. Booting up

Connect serial cable to APU4D4 and USB convertor. Make sure TXD and RXD lines are crossed ([null modem cable](https://en.wikipedia.org/wiki/Null_modem)). Fire up the serial console on your computer:

```
minicom -D /dev/ttyUSB0
```

Hit Ctrl+A and P to open comm parameters dialog. APU boards use 115200 8N1 by default. Also make sure you disable HW flow control (that's where I got stuck for a while): hit Ctrl+A and O, select "Serial port setup" and make sure "Hardware Flow Control" is set to "No".

Now we're ready to power up! LEDs should indicate that APU is alive. Your console should print some BIOS messages.

## 3. Installing Archlinux

Our OS of choice will be Archlinux. While probably not the best option for production servers, it's minimal size and simplicity makes it appealing. So head to [download page](https://archlinux.org/download/) and download iso. Copy to USB stick (/dev/sdX should be replaced by correct device file).

```
sudo dd if=Downloads/archlinux-2021.07.01-x86_64.iso of=/dev/sdX
```

Insert USB stick into APU. Power-cycle the APU and wait. USB stick should be first boot option by default (if not, select boot device using boot menu - F10 key). After bootloader starts, hit "Tab" and add following parameters to boot command:

```
console=ttyS0,115200n8
```

Note that console output might be broken and won't echo your characters correctly. Nevermind that, type it all in, hit enter and wait for login prompt to show up. Go through installation (using [this guide](https://wiki.archlinux.org/title/Installation_guide))

Note that when installing bootloader ([syslinux](https://wiki.archlinux.org/title/Syslinux) in my case), it is necessary to add `console=ttyS0,115200n8` to kernel boot parameters in bootloader config file to make kernel use serial console. 

Last thing before we reboot - while in `chroot` environment, install `dhclient` using

```
pacman -Sy
pacman -S dhclient
```

Otherwise our fresh install would have no way of connecting to the outside world.

Reboot and check that OS works as expected (don't forget to remove usb stick with live arch).

# Network configuration

## Prerequisities

Install all necessary utilities:

```
pacman -Sy
pacman -S bridge-utils iw traceroute hostapd sshd
```

## Setting up WiFi link layer (hostapd)

Edit `/etc/hostapd/hostapd.conf`:

```
interface=wlp5s0
bridge=br0

# SSID to be used in IEEE 802.11 management frames
ssid=YOUR_SSID_HERE
# Driver interface type (hostap/wired/none/nl80211/bsd)
driver=nl80211
# Country code (ISO/IEC 3166-1) - change to your country code
country_code=CZ

# Operation mode (a = IEEE 802.11a (5 GHz), b = IEEE 802.11b (2.4 GHz)
hw_mode=g
# Channel number - choose one that has low traffic
channel=7
# Maximum number of stations allowed
max_num_sta=5

# Bit field: bit0 = WPA, bit1 = WPA2
wpa=2
# Bit field: 1=wpa, 2=wep, 3=both
auth_algs=1

# Set of accepted cipher suites; disabling insecure TKIP
wpa_pairwise=CCMP
# Set of accepted key management algorithms
wpa_key_mgmt=WPA-PSK
wpa_passphrase=YOUR_PASSWD_HERE

# hostapd event logger configuration
logger_stdout=-1
logger_stdout_level=2

# Uncomment and modify the following section if your device supports 802.11n
## Enable 802.11n support
#ieee80211n=1
## QoS support
#wmm_enabled=1
## Use "iw list" to show device capabilities and modify ht_capab accordingly
#ht_capab=[HT40+][SHORT-GI-40][TX-STBC][RX-STBC1][DSSS_CCK-40]
```

## Network management script

Since we don't want to use higher-level management tools (like NetworkManager) and our network scenario is static, we'll use simple startup script to config networking.

Fire up your favourite editor:

```
vim /root/network_config.sh
```

```
#!/usr/bin/bash

# variables
WL=wlp5s0

# wait until wireless interface exists
while true; do
        ip link set dev $WL up;
        if [[ $? == 0 ]]; then
                break
        fi
done

# start hostapd (WiFi link layer)
systemctl start hostapd

# bridge wifi and all ethernet interfaces
brctl addbr br0
brctl addif br0 enp1s0 enp2s0 enp3s0 enp4s0 
ip link set dev br0 up
ip link set dev enp1s0 up
ip link set dev enp2s0 up
ip link set dev enp3s0 up
ip link set dev enp4s0 up
ip link set dev $WL up

#
# FIREWALL
#

# reset tables
ebtables-nft -F
iptables -F

# WiFi whitelist - this is optional
# allow only whitelisted MACs (stored in file whitelist.db, one MAC per line)
# note that this is still vulnerable to MAC spoofing
for mac in $(cat /root/whitelist.db); do
        echo "Whitelisting $mac for wireless connection"
        ebtables-nft -A FORWARD -i $WL -s "$mac" -j ACCEPT
        ebtables-nft -A INPUT -i $WL -s "$mac" -j ACCEPT
done
# drop everything else on wireless (frames coming from non-whitelisted MACs)
ebtables-nft -A INPUT -i $WL -j DROP
ebtables-nft -A FORWARD -i $WL -j DROP

# obtain address for ourselves from eth master port (enp1s0)
dhclient br0
```

File `whitelist.db` should look like this (using dummy MAC addresses):

```
11:22:33:44:55:66
aa:bb:cc:dd:ee:ff
```

## Running script on startup

Create unit file in `/etc/systemd/system`:

```
[Unit]
Description=Network configuration script
After=hostapd.service
Requires=hostapd.service

[Service]
Type=oneshot
ExecStart=/root/network_config.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

Enable service:

```
systemctl enable network_config
```

## Enabling SSH

If you intend to only use root account on device, don't forget to set `PermitRootLogin yes` in `/etc/ssh/sshd_config`. Then enable and start ssh server using:

```
systemctl enable sshd
systemctl start sshd
```

For better security you may use public key authentication and disable password login to ssh.

## Local DNS resolution

Since device itself has dynamic IP address, having a local name would help. Local name resolution is done using [multicast DNS](https://en.wikipedia.org/wiki/Multicast_DNS). [Here](https://www.howtogeek.com/167190/how-and-why-to-assign-the-.local-domain-to-your-raspberry-pi/) is nice how-to guide. On Archlinux you need to install following packages:

```
pacman -S avahi
systemctl start avahi-daemon
systemctl enable avahi-daemon
``

# Notes

## Gogs

if deploying gogs:

* install gogs in /home/git
* place .ssh/authorized_keys to /srv/git
* for user git use bash instead of git-shell (in /etc/passwd)

in /home/git/gogs/custom/conf/app.ini:

* [repository] ROOT = /srv/git

# Resources

https://www.pcengines.ch/apu4d4.htm

