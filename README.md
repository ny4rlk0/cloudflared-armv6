# Cloudflared ARMv6 Build Repository

Unofficial builds for Raspberry Pi 1, Zero, and Zero W

[![Build Cloudflared](https://github.com/IndraGunawan/cloudflared-armv6/actions/workflows/build.yaml/badge.svg)](https://github.com/IndraGunawan/cloudflared-armv6/actions/workflows/build.yaml)

## Build Repository Only

This repo **ONLY PROVIDES BINARIES FOR ARMv6 DEVICES AND DOES NOT ALTER SOURCE CODE** for the Cloudflared repo. For software issues regarding Cloudflared, go to the [Cloudflared repo](https://github.com/cloudflare/cloudflared) and open an issue there.


Why use DNS-Over-HTTPS? 1¶
DNS-Over-HTTPS is a protocol for performing DNS lookups via the same protocol you use to browse the web securely: HTTPS.

With standard DNS, requests are sent in plain-text, with no method to detect tampering or misbehavior. This means that not only can a malicious actor look at all the DNS requests you are making (and therefore what websites you are visiting), they can also tamper with the response and redirect your device to resources in their control (such as a fake login page for internet banking).

DNS-Over-HTTPS prevents this by using standard HTTPS requests to retrieve DNS information. This means that the connection from the device to the DNS server is secure and can not easily be snooped, monitored, tampered with or blocked. It is worth noting, however, that the upstream DNS-Over-HTTPS provider will still have this ability.

Configuring DNS-Over-HTTPS¶
Along with releasing their DNS service 1.1.1.1, Cloudflare implemented DNS-Over-HTTPS proxy functionality into one of their tools: cloudflared.

In the following sections, we will be covering how to install and configure this tool on Pi-hole.

Info

The cloudflared binary will work with other DoH providers (for example, you could use https://8.8.8.8/dns-query for Google's DNS-Over-HTTPS service).

Installing cloudflared¶
The installation is fairly straightforward, however, be aware of what architecture you are installing on (amd64 or arm).

AMD64 architecture (most devices)¶
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
armhf architecture (32-bit Raspberry Pi)¶
Here we are downloading the precompiled binary and copying it to the /usr/local/bin/ directory to allow execution by the cloudflared user. Proceed to run the binary with the -v flag to check it is all working:
```
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm
sudo mv -f ./cloudflared-linux-arm /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared
cloudflared -v
```
Info

Users have reported that the current version of cloudflared produces a segmentation fault error on Raspberry Pi Zero W, Model 1B and 2B. Currently, there is no known workaround.
```
arm64 architecture (64-bit Raspberry Pi)¶
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64
sudo mv -f ./cloudflared-linux-arm64 /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared
cloudflared -v
```
cloudflared archive page¶
You can find all cloudflared binary releases on https://github.com/cloudflare/cloudflared/releases.

Configuring cloudflared to run on startup¶
Create a cloudflared user to run the daemon:
```
sudo useradd -s /usr/sbin/nologin -r -M cloudflared
```
Proceed to create a configuration file for cloudflared:
```
sudo nano /etc/default/cloudflared
```
Edit configuration file by copying the following in to /etc/default/cloudflared. This file contains the command-line options that get passed to cloudflared on startup:

# Commandline args for cloudflared, using Cloudflare DNS
```
CLOUDFLARED_OPTS=--port 5053 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query
```
Update the permissions for the configuration file and cloudflared binary to allow access for the cloudflared user:
```
sudo chown cloudflared:cloudflared /etc/default/cloudflared
sudo chown cloudflared:cloudflared /usr/local/bin/cloudflared
```
Then create the systemd script by copying the following into /etc/systemd/system/cloudflared.service. This will control the running of the service and allow it to run on startup:
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
#Set it up
```
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```
Now test that it is working! Run the following dig command, a response should be returned similar to the one below:

pi@raspberrypi:~ $ dig @127.0.0.1 -p 5053 google.com
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
Configuring Pi-hole¶
Finally, configure Pi-hole to use the local cloudflared service as the upstream DNS server by specifying 127.0.0.1#5053 as the Custom DNS (IPv4):
