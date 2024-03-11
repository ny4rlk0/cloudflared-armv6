# Cloudflared ARMv6 Build Repository

Unofficial builds for Raspberry Pi 1, Zero, and Zero W

[![Build Cloudflared](https://github.com/IndraGunawan/cloudflared-armv6/actions/workflows/build.yaml/badge.svg)](https://github.com/IndraGunawan/cloudflared-armv6/actions/workflows/build.yaml)

## Build Repository Only

This repo **ONLY PROVIDES BINARIES FOR ARMv6 DEVICES AND DOES NOT ALTER SOURCE CODE** for the Cloudflared repo. For software issues regarding Cloudflared, go to the [Cloudflared repo](https://github.com/cloudflare/cloudflared) and open an issue there.


## Why use DNS-Over-HTTPS?
DNS-Over-HTTPS is a protocol for performing DNS lookups via the same protocol you use to browse the web securely: HTTPS.

With standard DNS, requests are sent in plain-text, with no method to detect tampering or misbehavior. This means that not only can a malicious actor look at all the DNS requests you are making (and therefore what websites you are visiting), they can also tamper with the response and redirect your device to resources in their control (such as a fake login page for internet banking).

DNS-Over-HTTPS prevents this by using standard HTTPS requests to retrieve DNS information. This means that the connection from the device to the DNS server is secure and can not easily be snooped, monitored, tampered with or blocked. It is worth noting, however, that the upstream DNS-Over-HTTPS provider will still have this ability.

## Configuring DNS-Over-HTTPS
Along with releasing their DNS service 1.1.1.1, Cloudflare implemented DNS-Over-HTTPS proxy functionality into one of their tools: cloudflared.

In the following sections, we will be covering how to install and configure this tool on Pi-hole.

## Info

The cloudflared binary will work with other DoH providers (for example, you could use https://8.8.8.8/dns-query for Google's DNS-Over-HTTPS service).

## Installing cloudflared
The installation is fairly straightforward, however, be aware of what architecture you are installing on (amd64 or arm).

## AMD64 architecture (most devices)
Download the installer package, then use apt-get to install the package along with any dependencies. Proceed to run the binary with the -v flag to check it is all working:

# For Debian/Ubuntu
```
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo apt-get install ./cloudflared-linux-amd64.deb
cloudflared -v
```
# For CentOS/RHEL/Fedora
```
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-x86_64.rpm
sudo yum install ./cloudflared-linux-x86_64.rpm
cloudflared -v
```
# armhf architecture (32-bit Raspberry Pi OS Legacy, Pi Zero)
Here we are downloading the precompiled binary and copying it to the /usr/local/bin/ directory to allow execution by the cloudflared user. Proceed to run the binary with the -v flag to check it is all working:
```
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm
wget https://github.com/IndraGunawan/cloudflared-armv6/releases/download/2024.2.1-10/cloudflared
wget https://github.com/ny4rlk0/cloudflared-armv6/releases/download/2024.2.1-10/cloudflared
sudo mv -f ./cloudflared /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared
cloudflared -v
```
# arm64 architecture (64-bit Raspberry Pi)
Users have reported that the current version of cloudflared produces a segmentation fault error on Raspberry Pi Zero W, Model 1B and 2B. Currently, there is no known workaround.
```
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64
sudo mv -f ./cloudflared-linux-arm64 /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared
cloudflared -v
```
# cloudflared archive page
You can find all cloudflared binary releases on https://github.com/cloudflare/cloudflared/releases.

# Configuring cloudflared to run on startup:

# Create a cloudflared user to run the daemon:

`sudo useradd -s /usr/sbin/nologin -r -M cloudflared`

# Proceed to create a configuration file for cloudflared:

`sudo nano /etc/default/cloudflared`

Edit configuration file by copying the following in to /etc/default/cloudflared. This file contains the command-line options that get passed to cloudflared on startup:

# Commandline args for cloudflared, using Cloudflare DNS (Paste this while editing cloudflared)
```
CLOUDFLARED_OPTS=--port 5053 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query
```
# Update the permissions for the configuration file and cloudflared binary to allow access for the cloudflared user:
```
sudo chown cloudflared:cloudflared /etc/default/cloudflared
sudo chown cloudflared:cloudflared /usr/local/bin/cloudflared
```
# Then create the systemd script by copying the following into /etc/systemd/system/cloudflared.service. This will control the running of the service and allow it to run on startup:
`sudo nano /etc/systemd/system/cloudflared.service`
```
sudo nano /etc/systemd/system/cloudflared.service
[Unit]
Description=cloudflared DNS over HTTPS proxy
After=syslog.target network-online.target

[Service]
Type=simple
User=cloudflared
EnvironmentFile=/etc/default/cloudflared
ExecStart=/usr/local/bin/cloudflared proxy-dns $CLOUDFLARED_OPTS
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
Enable the systemd service to run on startup, then start the service and check its status:
```
# Set it up
```
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```
# Now test that it is working! Run the following dig command, a response should be returned similar to the one below:

`pi@raspberrypi:~ $ dig @127.0.0.1 -p 5053 google.com`
```
; <<>> DiG 9.11.5-P4-5.1-Raspbian <<>> @127.0.0.1 -p 5053 google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12157
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 22179adb227cd67b (echoed)
;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             191     IN      A       172.217.22.14

;; Query time: 0 msec
;; SERVER: 127.0.0.1#5053(127.0.0.1)
;; WHEN: Wed Dec 04 09:29:50 EET 2019
;; MSG SIZE  rcvd: 77
```
# if you don't have dig. Install it.
`pi@raspberrypi:~ $ sudo apt-get install dnsutils`
# Configuring Pi-hole
Finally, configure Pi-hole to use the local cloudflared service as the upstream DNS server by specifying 127.0.0.1#5053 as the Custom DNS (IPv4):

`sudo nano /etc/dhcpcd.conf`

Then change your settings accordingly. I am assuming you are using ethernet cable to connected to Wi-Fi router.
If not use your selected interface. You can query your network interface with 

```
pi@raspberrypi:~ $ ip r | grep default
default via 192.168.1.1 dev eth0 src 192.168.1.3 metric 202
```
See mine is eth0 if you are using different interface, and pi zero ip, router ip (modem) change settings below accordingly. 
#Example /etc/dhcpcd.conf
```
# A sample configuration for dhcpcd.
# See dhcpcd.conf(5) for details.

# Allow users of this group to interact with dhcpcd via the control socket.
#controlgroup wheel

# Inform the DHCP server of our hostname for DDNS.
hostname

# Use the hardware address of the interface for the Client ID.
clientid
# or
# Use the same DUID + IAID as set in DHCPv6 for DHCPv4 ClientID as per RFC4361.
# Some non-RFC compliant DHCP servers do not reply with this set.
# In this case, comment out duid and enable clientid above.
#duid

# Persist interface configuration when dhcpcd exits.
persistent

# Rapid commit support.
# Safe to enable by default because it requires the equivalent option set
# on the server to actually work.
option rapid_commit

# A list of options to request from the DHCP server.
option domain_name_servers, domain_name, domain_search, host_name
option classless_static_routes
# Respect the network MTU. This is applied to DHCP routes.
option interface_mtu

# Most distributions have NTP support.
#option ntp_servers

# A ServerID is required by RFC2131.
require dhcp_server_identifier

# Generate SLAAC address using the Hardware Address of the interface
#slaac hwaddr
# OR generate Stable Private IPv6 Addresses based from the DUID
slaac private

# Example static IP configuration:
#interface eth0
#static ip_address=192.168.0.10/24
#static ip6_address=fd51:42f8:caae:d92e::ff/64
#static routers=192.168.0.1
#static domain_name_servers=192.168.0.1 8.8.8.8 fd51:42f8:caae:d92e::1

# It is possible to fall back to a static IP if DHCP fails:
# define static profile
#profile static_eth0
#static ip_address=192.168.1.23/24
#static routers=192.168.1.1
#static domain_name_servers=192.168.1.1

# fallback to static profile on eth0
#interface eth0
#fallback static_eth0
interface eth0
static ip_address=192.168.1.3/24
static routers=192.168.1.1
static domain_name_servers=127.0.0.1#5053
```
# Install Pi-Hole to use DOH another computers.
`curl -sSL https://install.pi-hole.net |sudo bash`

Select I already have static ip.
Select I wanna use Custom DNS.
Type this.

`127.0.0.1#5053`

Note your pi-hole password.

Now set your router DNS adresses to `192.168.1.3` or another one if you changed that ip.
Now you have set up a DOH server that queries DNS domains over HTTPS from `Cloudflare`.
You can access your pi-hole from `192.168.1.3/admin` and `password` you got while last page of the setup screen.
