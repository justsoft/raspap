### This is a dedicated reference for creating an Access Point ( bridge ) by Raspberry Pi.
#### The original information can be found here: https://raspberrypi.stackexchange.com/questions/88214/setting-up-a-raspberry-pi-as-an-access-point-the-easy-way
### ♦ General setup
#### Switch over to systemd-networkd
For detailed information look at ([1](https://raspberrypi.stackexchange.com/a/78788/79866)). Here only in short. Execute these commands:
```
# disable classic networking
rpi ~$ sudo -Es
rpi ~# systemctl mask networking.service
rpi ~# systemctl mask dhcpcd.service
rpi ~# mv /etc/network/interfaces /etc/network/interfaces~
rpi ~# sed -i '1i resolvconf=NO' /etc/resolvconf.conf

# enable systemd-networkd
rpi ~# systemctl enable systemd-networkd.service
rpi ~# systemctl enable systemd-resolved.service
rpi ~# ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```
#### Configure wpa_supplicant as access point
To configure wpa_supplicant as access point create this file with your settings for ` country= `, ` ssid= `, ` psk= ` and maybe ` frequency= `. You can just copy and paste this in one block to your command line beginning with ` cat ` and including both EOF (delimiter EOF will not get part of the file):

```
rpi ~# cat > /etc/wpa_supplicant/wpa_supplicant-wlan0.conf <<EOF
country=DE
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="RPiNet"
    mode=2
    key_mgmt=WPA-PSK
    proto=RSN WPA
    psk="password"
    frequency=2437
}
EOF

rpi ~# chmod 600 /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
rpi ~# systemctl disable wpa_supplicant.service
rpi ~# systemctl enable wpa_supplicant@wlan0.service
```
### ♦ Setting up an access point with a bridge
Example for this setup:
```
                               RPi
               wifi   ┌──────bridge──────┐   wired            wan
mobile-phone <.~.~.~> │(wlan0) br0 (eth0)│ <-------> router <-----> INTERNET
            \                   |                   / DHCP-server
           (dhcp)             (dhcp)          192.168.50.1
```
If you have already an ethernet network with DHCP server and internet router and you want to expand it with a wifi access point but with the same ip addresses then you use a bridge. This is often used as an uplink to a router.
#### Setup
Do the first part of "**General setup**" then clean up directory /etc/systemd/network but don't touch ` 99-default.link ` if present. Create the following three files to configure **eth0** and **br0**. The ip addresses are examples. You have to set your own.
```
rpi ~# cat > /etc/systemd/network/02-br0.netdev <<EOF
[NetDev]
Name=br0
Kind=bridge
EOF

rpi ~# cat > /etc/systemd/network/04-br0_add-eth0.network <<EOF
[Match]
Name=eth0
[Network]
Bridge=br0
EOF

rpi ~# cat > /etc/systemd/network/12-br0_up.network <<EOF
[Match]
Name=br0
[Network]
DHCP=yes
# to use static IP uncomment these and comment DHCP=yes
#Address=192.168.50.60/24
#Gateway=192.168.50.1
#DNS=84.200.69.80 84.200.70.40
EOF
```
Now we have to tell wpa_supplicant to use a bridge. We do it by modifying its service with:
```
rpi ~# systemctl edit wpa_supplicant@wlan0.service
```
In the empty editor insert these statements, save them and quit the editor:
```
[Service]
ExecStartPre=/sbin/iw dev %i set type __ap
ExecStartPre=/bin/ip link set %i master br0

ExecStart=
ExecStart=/sbin/wpa_supplicant -c/etc/wpa_supplicant/wpa_supplicant-%I.conf -Dnl80211,wext -i%I -bbr0

ExecStopPost=-/bin/ip link set %i nomaster
ExecStopPost=-/sbin/iw dev %i set type managed
```
Reboot.
That's it.

### WPA2 with AES
The previous configure will let the AP works as WPA-PSK mode, to swith to WPA2-PSK & AES encryption:
```
country=CA
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="Mountain-2g"
    psk="<The_shared_key_for_this_AP>"
    mode=2
    proto=RSN
    key_mgmt=WPA-PSK
    pairwise=CCMP
    group=CCMP
    auth_alg=OPEN
}
```
### Control the wlan0 TX power
The default TX power is 20 dBm, lower down the TX power if you just need an AP in a small room.
```
/sbin/iwconfig wlan0 txpower 0
```
