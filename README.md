# Forked from https://github.com/cuddlyconcretepangolin/rhf2s308_experience
# credits to him


# rhf2s308_experiences
Experiences dealing with rhf2s308 for helium mining. 

I am documenting this for my own use (when inevitably I need to flash the firmware and start over).  I would not recommend anyone execute these commands unless you know what you are doing. I take no responsibility if you mess up your miner.

# Obtaining shell access

It seems that RisingHF has begun to lock us out of the devices.  The credentials in the manual (https://risinghf-official-website.oss-cn-shenzhen.aliyuncs.com/static/file/product/GX5CI5KeGI7Dl7fFvTrLrA==.pdf) are stated as username/password: `rxhf`/`risinghf`.  These work for a few moments after a firmware flash, but suddenly change moments later. A solution is to quickly create your own username and password, and give yourself sudo access.

1) Flash firmware (https://risinghf-pkg.oss-cn-shenzhen.aliyuncs.com/risinghf/firmware/rhf2s308-1.0.4-20220107-helium.img.zip) or (https://risinghf-pkg.oss-cn-shenzhen.aliyuncs.com/risinghf/firmware/RHF2s308_firmware%20_1.0.7_20220125.zip)
2) Unplug USB.
3) Disconnect power.
4) Reconnect power.
6) Immediately start attempting to log in with `rxhf`/`risinghf` username/password combo (via serial (i.e. USB). Use PuTTY on Windows. Refer to the manual above for this as it's accurate this far). Remember that this is Linux so the password field will not show placeholders as you type.
8) As soon as you're logged in, create a new user (username is your choice): `sudo adduser username`
9) Then give your new account sudo access: `sudo adduser username sudo`

(Aside: several moments later, if you try to log in with `rxhf/risinghf` you likely will not be able to.  If you don't want a user account with mystery credentials on *your own hardware*, then you can change the password with `sudo pw rxhf` since you now have your own account with root access).

## Turn on `ssh` access

Run `sudo openssh`.  It seems that the ssh service gets periodically masked again. I add
```
./usr/local/sbin/openssh & 
```
to my `/etc/rc.local` file to unmask on boot, or to crontab to run periodically.


# Accessing the web dashboard

``` sudo openweb``` will start the web server and give access on port 80 to the dashboard discussed in the manual. The information is generic, and mostly irrelevant to Helium mining.


# Enabling Wifi for Internet

The stock box only uses wifi for diagnostics/debugging/admin.  Thus it creates an access point. If you attempt to connect to Wifi using the Helium app, you might get lucky and do so before this service has created the access point, making it seem like it's working, but the hotspot wifi will not be consistent as the access point services will be constantly battling for access to the wlan0 network interface.  Here I attempt to disable this access point so that I can use the Wifi interface as an internet connection. 

First, we must disable the two services that initialize the access point:
``` bash
systemctl stop init_wifi
systemctl disable init_wifi
systemctl mask init_wifi

systemctl stop create_ap
systemctl disable create_ap
systemctl mask create_ap
```

The networking on this device is managed by `connman`, but if you try to connect to wifi using `connmanctl` right now, you will get a `No Carrier` error. This is because (for some reason) there are two instances of the `wpa_supplicant` service started. 

``` bash 
root@rhf2s308:# ps -aux | grep wpa_supplicant
root       536  0.0  0.1  12564  1704 ?        Ss   14:11   0:00 /sbin/wpa_supplicant -u -s -O /run/wpa_supplicant
root       795  0.0  0.0  12712   836 ?        Ss   14:11   0:00 wpa_supplicant -B -c/etc/wpa_supplicant/wpa_supplicant.conf -iwlan0 -Dnl80211,wext
```

The first one is launched by systemd (you can verify by comparing this command to the `ExecStart` entry in `/usr/lib/systemd/system/wpa_supplicant.service`) and the second is launched by dhcpcd via `/lib/dhcpcd/dhcpcd-hooks/10-wpa_supplicant`.  It sounds like this is an issue with Raspberry Pis in general (https://forums.raspberrypi.com/viewtopic.php?t=292401) and it is safe to disable the systemd service, as dhcpcd will take care of launching wpa_supplicant. In fact, if you look at the systemd service status (`systemctl status wpa_supplicant`), you might even see that it's full of complaints that the driver is already in use ("nl80211: kernel reports: Match already configured").  So we stop it and mask it as well:

``` bash 
systemctl stop wpa_supplicant
systemctl disable wpa_supplicant
systemctl mask wpa_supplicant
```

Now, all that's left to do is add our network credentials to the wpa_supplicant configuration file.  Modify `/etc/wpa_supplicant/wpa_supplicant.conf` to look like the following: (the first two lines should already be present; obviously put your own network's credentials. Use `key_mgmt=NONE` for unsecured Wifi (untested)). 
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
	ssid="MY-WIFI-NETWORK-NAME"
 	psk="My-Password"
}
```
After doing all this and rebooting, I was automatically connected to wifi when the system came back up:

``` bash 
username@rhf2s308:~ $ ifconfig
...
wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.163  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::84b9:6e67:6246:417e  prefixlen 64  scopeid 0x20<link>
        ether 20:50:e7:10:9d:4b  txqueuelen 1000  (Ethernet)
        RX packets 2600  bytes 2244628 (2.1 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2220  bytes 467805 (456.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
...

```

My concern was that the Helium miner would have the `eth0` interface hard coded and would not pick up on `wlan0` being connected, but after pulling the ethernet plug, the `peer_book` went relayed as expected (`docker exec miner miner peer book -s`) as I do not have port forwarding set up for the Wifi IP.  Since reboot, I have successfully heard and submitted (over wifi) a witness, so it seems to be working:

```bash
root@rhf2s308:~# docker exec miner cat /var/data/log/console.log | grep --text witness

2022-01-26 15:54:17.375 8 [info] <0.1642.0>@miner_onion_server:decrypt:{372,13} could not decrypt packet received via radio: treating as a witness
2022-01-26 15:54:17.377 8 [info] <0.3142.0>@miner_onion_server:send_witness:{188,13} sending witness at RSSI: -99, Frequency: 904.7, SNR: -2.75
2022-01-26 15:54:17.573 8 [warning] <0.3142.0>@miner_onion_server:send_witness:{243,37} failed to dial challenger "/p2p/redacted": not_found
2022-01-26 15:54:47.592 8 [info] <0.3142.0>@miner_onion_server:send_witness:{246,37} re-sending witness at RSSI: -99, Frequency: 904.7, SNR: -2.75
2022-01-26 15:54:49.193 8 [info] <0.3142.0>@miner_onion_server:send_witness:{251,37} successfully sent witness to challenger "/p2p/redacted" with RSSI: -99, Frequency: 904.7, SNR: -2.75
```

(i.e. it heard a witness, then ~30 seconds later was successful in dialing the challenger)


## Frozen sync
On Wednesday, Feb 2, my miner started to fall behind the blockchain and the logs were full of messages along the lines of `no plausible blocks in batch`. I do not have a fix for this; I do not know why it happened, but I have a workaround. It amounts to installing the miner on another computer, letting it sync, then copying over the entire blockchain directory. This worked for me, and my miner is back witnessing and earning rewards.  I will note that if you don't have some basic knowledge of Unix, this is probably not going to be possible for you to undertake, as it's not a straight copy/paste of my commands.

You need another computer (not sure requirements, but I used a Ubuntu server I have at home).  Using this other computer (I'll call it "B"):
1) Install the Helium miner: https://github.com/helium/devdocs/blob/master/blockchain/run-your-own-miner.md.
2) Start the miner and let it sync.  It will download a snapshot and sync. Mine took about 4 hours, but it's a pretty powerful computer. It was much faster than the low-powered hotspot. Keep an eye on the logs and move to 3) once it's caught up.
3) Once it's caught up, stop the miner. On B, run: `sudo docker stop miner`.
4) Stop the miner on the RisingHF hotspot: `sudo docker stop miner`.
5) Remove the miner data directory: On the hotspot: `sudo rm -r /opt/helium/miner_data`
6) Copy over the contents of `miner_data` folder from Computer B to hotspot. This exact command depends on where you chose to locally store the miner data. In the link in step (1) above, they chose `~/miner_data`. I used rsync to copy over the network. I ran the following command _from the hotspot_ as root is needed to write to `/opt/helium`. On the hotspot: 
```bash 
# user = username on Computer B 
# 192.168.0.333 = Local IP Address of Computer B (yours will be different)

rsync -av --info=progress2 user@192.168.0.333:/home/user/miner_data/ /opt/helium/miner_data
```
7) Start the miner on the hotspot: ```sudo docker start miner```
8) The hot spot should now only be several minutes behind (however long the copy operation took). 


The nice thing about this solution is that now I have another miner running on Computer B (that I will just allow to continue running so it's always in sync), and I can copy over the blockchain if it ever falls behind again, or if I need to resync firmware in the future.




#
#
#
#
#
#
#

## coloured_witness_log.sh

```bash 

#!/bin/bash

# This will tail the miner log and output a line every time
# 1) A witness is heard
# 2) A witness is successfully submitted to the challenger, and
# 3) The maximum number of retries is met

# For example: 
# 2022-01-26 17:03:07.736 Heard witness
# 2022-01-26 17:03:38.994 Successfully sent 
# 2022-01-26 17:03:43.682 Max retry

#process everything in the log as of now:
sudo docker exec miner cat     /var/data/log/console.log | grep --text witness |  awk '{if ($6=="successfully") print "\x1b[32m"$1" "$2" Successfully sent \x1b[0m"; else if ($13=="treating") print $1" "$2" Heard witness"; else if ($10=="max") print "\x1b[31m"$1" "$2" Max retry\x1b[0m"; }'
#tail the log
sudo docker exec miner tail -f /var/data/log/console.log | grep --text witness |  awk '{if ($6=="successfully") print "\x1b[32m"$1" "$2" Successfully sent \x1b[0m"; else if ($13=="treating") print $1" "$2" Heard witness"; else if ($10=="max") print "\x1b[31m"$1" "$2" Max retry\x1b[0m"; }'


```

## Status of miner or Miner General Info

```bash 

sudo docker exec miner miner info summary

```

```
********************
Miner General Info
********************

+-------------+------------------------------------------------------------------------------------+
|    name     |                                       result                                       |
+-------------+------------------------------------------------------------------------------------+
| miner name  |                                deep-charcoal-cobra                                 |
|mac addresses|                                wwan0 : XXXXXXXXXXXX                                |
|             |                               docker0 : XXXXXXXXXXXX                               |
|             |                                eth0 : XXXXXXXXXXXX                                 |
|             |                                wlan0 : XXXXXXXXXXXX                                |
|             |                                 lo : 000000000000                                  |
|  block age  |                                        112                                         |
|    epoch    |                                       32695                                        |
|   height    |                                      1232499                                       |
| sync height |                                      1232499                                       |
|   uptime    |                      0 Days, 0 Hours, 53 Minutes, 25 Seconds                       |
|peer book siz|                                       20898                                        |
|firmware vers|           cat: can't open '/etc/lsb_release': No such file or directory            |
|gateway detai|Gateway location: 631426055126266367Owner address: /p2p/1342GXoRx4GjcguX5KZ9qj8XegnL|
+-------------+------------------------------------------------------------------------------------+


********************
Miner P2P Info
********************

+---------+---------+
|  name   | result  |
+---------+---------+
|connected|   yes   |
|dialable |   yes   |
|nat_type |symmetric|
| height  | 1232500 |
+---------+---------+


********************
Miner log errors
********************

***** Transaction related errors *****

: 
***** POC related errors         *****

: 
***** General errors             *****

: 

```
## Checking the P2P Network
```
docker exec miner miner peer book -s


+--------------------------------------------------------+------------------------------+------------+-----------+---------+------------+
|                        address                         |             name             |listen_addrs|connections|   nat   |last_updated|
+--------------------------------------------------------+------------------------------+------------+-----------+---------+------------+
|/p2p/11dwT67atkEe1Ru6xhDqPhSXKXmNhWf3ZHxX5S4SXhcdmhw3Y1t|{ok,"genuine-steel-crocodile"}|     2      |     4     |symmetric|   3.148s   |
+--------------------------------------------------------+------------------------------+------------+-----------+---------+------------+

+----------------------------------------------------------------------------------------------------------------------------+
|                                                 listen_addrs (prioritized)                                                 |
+----------------------------------------------------------------------------------------------------------------------------+
|/p2p/11apmNb8phR7WXMx8Pm65ycjVY16rjWw3PvhSeMFkviWAUu9KRD/p2p-circuit/p2p/11dwT67atkEe1Ru6xhDqPhSXKXmNhWf3ZHxX5S4SXhcdmhw3Y1t|
|                                                 /ip4/192.168.3.6/tcp/36397                                                 |
+----------------------------------------------------------------------------------------------------------------------------+

+--------------------------+-----------------------------+---------------------------------------------------------+------------------------------+
|          local           |           remote            |                           p2p                           |             name             |
+--------------------------+-----------------------------+---------------------------------------------------------+------------------------------+
|/ip4/192.168.3.6/tcp/36397|/ip4/104.248.122.141/tcp/2154|/p2p/112GZJvJ4yUc7wubREyBHZ4BVYkWxQdY849LC2GGmwAnv73i5Ufy|{ok,"atomic-parchment-snail"} |
|/ip4/192.168.3.6/tcp/36397| /ip4/73.15.36.201/tcp/13984 |/p2p/112MtP4Um2UXo8FtDHeme1U5A91M6Jj3TZ3i2XTJ9vNUMawqoPVW|   {ok,"fancy-glossy-rat"}    |
|/ip4/192.168.3.6/tcp/36397| /ip4/24.5.52.135/tcp/41103  |/p2p/11AUHAqBatgrs2v6j3j75UQ73NyEYZoH41CdJ56P1SzeqqYjZ4o |  {ok,"skinny-fuchsia-mink"}  |
|/ip4/192.168.3.6/tcp/46059| /ip4/34.222.64.221/tcp/2154 |/p2p/11LBadhdCmwHFnTzCTucn6aSPieDajw4ri3kpgAoikgnEA62Dc6 | {ok,"skinny-lilac-mustang"}  |
+--------------------------+-----------------------------+---------------------------------------------------------+------------------------------+
```

## ZeroTier Install

follow this guide https://mangolassi.it/topic/16204/how-to-install-zerotier-on-a-raspberry-pi/5

```
curl - https://install.zerotier.com/ sudo bash

sudo zerotier-cli join xxxxx
```
If error:
Error connecting to the Zero Tier service:
Please check that the service is running and that TCP port 9993 can be contacted
via 127.0.0.1.
```
sudo iptables -D POSTROUTING -t nat -j MASQUERADE
delete /var/lib/zerotier-one/identity .public and /var/lib/zerotier-one/identity.secret
```
for new address


## 

sudo docker exec miner miner info height


zram install ??
https://pimylifeup.com/raspberry-pi-zram/


sudo docker exec miner miner versions


lsblk

df -h

du -h /opt/helium/miner_data


Cpied from discord

I think I traced the reboot issue down!

Linked it to a RisingHF service called abnormal_det that I'm theorizing tries to reboot the miner if eth0 is plugged in and it can't get an IP in 2 mins

So to buy some time you can run the following to disable that service.

`sudo systemctl stop abnormal_det`
`sudo systemctl disable abnormal_det`
`sudo systemctl mask abnormal_det`

My eth0 is still not working properly. Luckily I can force wlan0 to client mode (https://github.com/cuddlyconcretepangolin/rhf2s308_experience) and use it that way. Also gives you time to enable ssh for later
