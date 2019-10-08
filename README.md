# Unifipi

## Install Unifi

- [Link lazyadmin.nl](https://lazyadmin.nl/home-network/installing-unifi-controller-on-a-raspberry-pi-in-5-min/)
- [Link forum unifi](https://community.ui.com/questions/Step-By-Step-Tutorial-Guide-Raspberry-Pi-with-UniFi-Controller-and-Pi-hole-from-scratch-headless/e8a24143-bfb8-4a61-973d-0b55320101dc)

### Start with updates
`sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get autoremove && sudo apt-get autoclean`

### Update Java 8
This step is pretty important with the newer release of the Unifi Controller. Starting with version 5.10.x the old Java version that comes with Raspbian isn’t supported anymore. So we are going to replace it with OpenJDK 8.

`sudo apt-get install openjdk-8-jre-headless -y`

### Install haveged
Not really necessary, but the startup of the Unifi Controller can take a bit long on a Raspberry Pi due to the fact there is no user interaction. Now you Pi should be running 24/7, so it’s not a big deal, but we can speed it up anyway.

Install haveged to solve the issue with the following cmd:

`sudo apt-get install haveged -y`

### Split memory with
If you run the Raspberry Pi without a monitor connected you can safely reduce the amount of ram used by the GPU. By default, the Pi has assigned 64MB RAM to the GPU. Because we are using CLI (Command Line Interface) to work on the Pi, we can reduce that amount for RAM, leaving more RAM available for the Pi self.

Open the Raspberry Pi config with the following command:

`sudo raspi-config`

1. Select **7. Advanced Options**
2. **A3 Memory Split**
3. Change 64 to **16MB**
4. Hit **Enter** and use **Esc** to close the config screen.

Make sure you reboot the Pi to apply the changes.

#### 1. Add the repository for the Unifi Controller
All Linux distros come with a source list, repository, of available packages to install. Unifi Controller is not listed in the default repositories, so we need to add it first:

``echo 'deb http://www.ubnt.com/downloads/unifi/debian stable ubiquiti' | sudo tee /etc/apt/sources.list.d/100-ubnt-unifi.list``
 

#### 2. Authenticate the software
To install the Unifi Controller software we need to authenticate the software that it’s the real software from Ubiquiti. This can be done with a gpg key so we can authenticate the software.

``sudo wget -O /etc/apt/trusted.gpg.d/unifi-repo.gpg https://dl.ubnt.com/unifi/unifi-repo.gpg``
 

#### 3. Install the Unifi Controller
We now have added the software to our list of available software and have the ability to check its authenticity. So let’s download the software and install the Unifi Controller on the Raspberry Pi:

``sudo apt-get update; sudo apt-get install unifi -y``


#### 4. Cleanup MongoDB
The installation may take a couple of minutes to complete. When done, we need to remove the default database that comes with MongoDB instance. This would only waste resources of our Pi, so we get rid of it:

```
sudo systemctl stop mongodb 
sudo systemctl disable mongodb
```

Go to your UniFi Controller via the IP address and port **https://192.168.x.x:8443**


## Install [Pi-hole](https://pi-hole.net/) with cloudfared DNS

[Link scotthelme blog](https://scotthelme.co.uk/securing-dns-across-all-of-my-devices-with-pihole-dns-over-https-1-1-1-1/)

#### Installing cloudflared
We're not going to use the full potential of cloudflared but it does everything we need. We're going to use it as a DoH proxy and the first task is to get it installed and running.

````
cd ~
wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-arm.tgz
mkdir argo-tunnel
tar -xvzf cloudflared-stable-linux-arm.tgz -C ./argo-tunnel
rm cloudflared-stable-linux-arm.tgz
cd argo-tunnel
./cloudflared --version
````

#### Start cloudflared with the right config

`sudo ./cloudflared proxy-dns --port 54 --upstream https://1.1.1.1/.well-known/dns-query --upstream https://1.0.0.1/.well-known/dns-query`

#### Run cloudflared as a service

If everything is all setup and running just fine the last step is to make sure cloudflared is always running. To do this we will create a systemd unit file to make sure of that.

`sudo nano /etc/systemd/system/dnsproxy.service`
````
[Unit]
Description=CloudFlare DNS over HTTPS Proxy
Wants=network-online.target
After=network.target network-online.target
 
[Service]
ExecStart=/home/pi/argo-tunnel/cloudflared proxy-dns --port 54 --upstream https://1.1.1.1/.well-known/dns-query --upstream https://1.0.0.1/.well-known/dns-query
Restart=on-abort
 
[Install]
WantedBy=multi-user.target
````

To ensure cloudflared runs on startup you can enable it with the following.

`sudo systemctl enable dnsproxy.service`

#### Install Pi-Hole

`curl -sSL https://install.pi-hole.net | bash`

Follow the setup process and fill in the values as you're asked along the way. It doesn't matter what default DNS service you use as we will be overwriting it soon. 

After install use this command to set a new password for the web interface:

`pihole -a -p`

The web interface is reachable via **https://192.168.x.x/admin**

Once the setup is done we only need to make changes to two files after I've looked over the instructions set out by [Olier](https://oliverhough.cloud/blog/configure-pihole-with-dns-over-https/) in this blog. First we need to edit a config file and remove the two instances of `server`.


`sudo nano /etc/dnsmasq.d/01-pihole.conf`

Comment out both server declarations, `#server=1.1.1.1` and `#server=1.0.0.1`, and replace them with our own `server=127.0.0.1#54`. After that we need to edit the next file.


`sudo nano /etc/pihole/setupVars.conf`

In here just comment out the 2 DNS addresses `#PIHOLE_DNS_1=1.1.1.1` and `#PIHOLE_DNS_2=1.0.0.1`. Once that's done you can restart the dnsmasq service with sudo systemctl restart dnsmasq.service and the Pi-Hole will now send DNS requests to cloudflared which is running as our DoH proxy.

Check if DoH works with:

- [1.1.1.1](https://1.1.1.1/help/)
- [Cloudfare Security check](https://www.cloudflare.com/ssl/encrypted-sni/)


## Install [PiVPN](http://www.pivpn.io/)

[pimylifeup guide](https://pimylifeup.com/raspberry-pi-vpn-server/)

`curl -L https://install.pivpn.io | bash`

After install, create a vpn user with 

`sudo pivpn add`

Once you press enter to these, the PiVPN script will tell Easy-RSA to generate the 2048-bit RSA private key for the client, and then store the file into `/home/pi/ovpns`.

**Don't forget to forward the port in your router as per the tutorial (default OpenVPN port: 1194 UDP).**

PiVPN does install and configure [Unnatended Upgrades](https://wiki.debian.org/UnattendedUpgrades)
We should add `ubiquiti` to it's blacklist so the controller is only updated manually (interactive installation and chance to do backup)

`sudo nano /etc/apt/apt.conf.d/50unattended-upgrades`

and add `"ubiquiti";` after the following lines:

````
// Python regular expressions, matching packages to exclude from upgrading
Unattended-Upgrade::Package-Blacklist {
    // The following matches all packages starting with linux-
//  "linux-";
    "ubiquiti";
````

#### Now verify that both of them work independently:

Connecting to the DNS server should blackhole common tracking domains such as google-analytics.com while allowing google.com)

Remotely connecting to the VPN should allow you to see your home IP (I tested it using OpenVPN for android and disabling wifi; that way I should have a different public IP than my desktop, due to being on the mobile network. After connecting to the VPN I had the same public IP as my desktop -> OpenVPN works correctly).

## Final config - OpenVPN through Pi-Hole
[marcstan guide](https://marcstan.net/blog/2017/06/25/PiVPN-and-Pi-hole/)

Now that OpenVPN and Pi-hole are both running independently it's time to connect them.

If you use OpenVPN from outside your network, you'll notice that it doesn't forward the DNS requests to Pi-Hole yet (e.g. google-analytics.com isn't blocked) so let's fix that:

Edit OpenVPN server config:
`sudo nano /etc/openvpn/server.conf`

Remove all existing `dhcp-option` entries you can find and add a single new one.

`push "dhcp-option DNS 10.8.0.1"`

(10.8.0.X being the default OpenVPN subnet, if you changed this in the installer, use your custom values).

Edit Pi-hole config:
`sudo nano /etc/pihole/setupVars.conf`

Add `PIHOLE_INTERFACE=tun0` below the `eth0`.

You should now have entries:
````
PIHOLE_INTERFACE=eth0
PIHOLE_INTERFACE=tun0
````

Edit one last file (the file should not yet exist):
`sudo nano /etc/dnsmasq.d/02-ovpn.conf`

Add single line `interface=tun0`

A file `01-pihole.conf` exists in the same directory and we could add this entry there, but we shouldn't. Edits to it may be overriden by any Pi-hole update.

Luckily, Pi-hole also respects config values from all `*.conf` files in the same directory and won't touch other files when updating, so this config should remain intact on future updates!

One final reboot: `sudo reboot now` and we're finished!

Now connect via OpenVPN and check if DoH works with:

- [1.1.1.1](https://1.1.1.1/help/)
- [Cloudfare Security check](https://www.cloudflare.com/ssl/encrypted-sni/)


## Bonus bloclists for Pi-hole

- (Amazing Blocklist dbl oist nl)[https://www.reddit.com/r/pihole/comments/bppug1/introducing_the/]
- (Firebog Blocklist Collection)[https://firebog.net]

## Connect to OpenVPN with MacOS

Use [Tunnelblick](https://tunnelblick.net/)

Add the profile `*.ovpn`
in `Settings` check [x] `Route all IPv4 traffic through the VPN`
in `Advanced`, check [x] `Allow changes to manually-set network settings`


## Backup the pi with `rpi-clone`

[rpi-clone Github repository](https://github.com/billw2/rpi-clone)

Install by cloning repository

```
cd ~/Downloads
git clone https://github.com/billw2/rpi-clone.git 
cd rpi-clone
sudo cp rpi-clone rpi-clone-setup /usr/local/sbin
```

Then use fdisk to get the backup SD card name

`sudo fdisk -l`

You should see the current SD card: sda or mmcblk0 most of the time
And below the backup SD card: sda or mmcblk1 generally

And finally, use rpi-clone

`sudo rpi-clone sda -v`

## Change PoE HAT Fan settings and turno off Wifi/Bluetooth

`sudo nano /boot/config.txt`
 
 and add the following lines
 
````
# POE Fan control
dtparam=poe_fan_temp0=58000,poe_fan_temp0_hyst=13000
dtparam=poe_fan_temp1=59000,poe_fan_temp1_hyst=2000

# Turn wifi and bluetooth off
#dtoverlay=disable-wifi
dtoverlay=disable-bt
````
In this example, the fan kick in at 58 degrees and reduce the temperature of 13 degrees (45)
if temp goes over 59 the fan will speed up till it reduce the temperature of 2 degrees (57) and goes back to normal speed.
The bluetooth is disable at boot but wifi not selected.





