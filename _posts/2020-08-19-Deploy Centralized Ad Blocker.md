---
layout: post
title: "Jump on DoH & Deploy a centralized Ad Blocker in your network"
excerpt: "How to avoid installing ad blocker on all you devices"
excerpt_separator: "<!--more-->"
categories:
  - security
tags:
  - linux
  - raspberry-pi
  - adblock
  - dns
last_modified_at: 2020-08-19T10:23:48-05:00
---

### Agaaain, It was DNS!

[DoH](https://en.wikipedia.org/wiki/DNS_over_HTTPS) :DNS over HTTPS allows you to encrypt the traffic from your PC to the DNS server. Encrypting this channel prevents your ISP from seeing the sites you visit, which is why Mozilla has been labeled an Internet villain!
Setup DoH with Firefox is very easy using CloudFlare or NextDNS

### Why use DNS-Over-HTTPS?

DNS-Over-HTTPS is a protocol for performing DNS lookups via the same protocol you use to browse the web securely: **HTTPS**.

With standard DNS, requests are sent in plain-text, with no method to detect tampering or misbehavior. This means that not only can a malicious actor look at all the DNS requests you are making (and therefore what websites you are visiting), they can also tamper with the response and redirect your device to resources in their control (such as a fake login page for internet banking).

DNS-Over-HTTPS prevents this by using standard HTTPS requests to retrieve DNS information. This means that the connection from the device to the DNS server is secure and can not easily be snooped, monitored, tampered with or blocked.
It is worth noting, however, that the upstream DNS-Over-HTTPS provider will still have this ability.

## DoH pro & Cons

* Bypass censorship on DNS queries, in one click
* Making it much more difficult for the ISP to maintain a navigation history
* complicate Man in The Middle Attacks

Be careful however, enabling this feature has a big disadvantage if you do it directly on a network device or browser: parental filters will mostly be disabled since they only intercept requested domain names and prohibit them if they are in their blacklist. As the content is then encrypted, they will no longer be able to do so.

## Solution: Pi-Hole

The Pi-hole® is a [DNS sinkhole](https://en.wikipedia.org/wiki/DNS_sinkhole) that protects your devices from unwanted content, without installing any client-side software.


```sh
curl -sSL https://install.pi-hole.net | bash
```
note: Piping bash can bother you, if you prefer to look at the code, here is the script to clone the Git repo:

```sh 
git clone --depth 1 https://github.com/pi-hole/pi-hole.git Pi-hole
cd "Pi-hole/automated install/"
sudo bash basic-install.sh
```
Once installed, configure your router to disable DHCP and promote your pi-hole system as DHCP server OR just configure the DNS to set your Pi-Hole Ip address.

Finally, you can just add those 2 magic lists:

* [StevenBlack/hosts](https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts)

* [malware list](https://mirror1.malwaredomains.com/files/justdomains)


You now have a strong DNSsinkhole system that catch all DNS queries.

![pihole](/images/pihole1.png)



## Configuring DNS-Over-HTTPS

Along with releasing their DNS service [1.1.1.1](https://blog.cloudflare.com/announcing-1111/), Cloudflare implemented DNS-Over-HTTPS proxy functionality into one of their tools: [`cloudflared`](https://github.com/cloudflare/cloudflared).

In the following sections, we will be covering how to install and configure this tool on `Pi-hole`.

### Installing `cloudflared`

The installation is fairly straightforward, however, be aware of what architecture you are installing on (`amd64` or `arm`).

#### AMD64 architecture (most devices)

Download the installer package, then use `apt-get` to install the package along with any dependencies. Proceed to run the binary with the `-v` flag to check it is all working:

```bash
# For Debian/Ubuntu
wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-amd64.deb
sudo apt-get install ./cloudflared-stable-linux-amd64.deb
cloudflared -v

# For CentOS/RHEL/Fedora
wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-amd64.rpm
sudo yum install ./cloudflared-stable-linux-amd64.rpm
cloudflared -v
```

**Note:** Binaries for other operating systems can be found here: <https://developers.cloudflare.com/argo-tunnel/downloads/>

#### ARM architecture (Raspberry Pi)

Here we are downloading the precompiled binary and copying it to the `/usr/local/bin/` directory to allow execution by the cloudflared user. Proceed to run the binary with the `-v` flag to check it is all working:

```bash
wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-arm.tgz
tar -xvzf cloudflared-stable-linux-arm.tgz
sudo cp ./cloudflared /usr/local/bin
sudo chmod +x /usr/local/bin/cloudflared
cloudflared -v
```

### Configuring `cloudflared` to run on startup

#### Manual way

Create a `cloudflared` user to run the daemon:

```bash
sudo useradd -s /usr/sbin/nologin -r -M cloudflared
```

Proceed to create a configuration file for `cloudflared`:

```bash
sudo nano /etc/default/cloudflared
```

Edit configuration file by copying the following in to `/etc/default/cloudflared`. This file contains the command-line options that get passed to cloudflared on startup:

```bash
# Commandline args for cloudflared
CLOUDFLARED_OPTS=--port 5053 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query
```

Update the permissions for the configuration file and `cloudflared` binary to allow access for the cloudflared user:

```bash
sudo chown cloudflared:cloudflared /etc/default/cloudflared
sudo chown cloudflared:cloudflared /usr/local/bin/cloudflared
```

Then create the `systemd` script by copying the following into `/etc/systemd/system/cloudflared.service`. This will control the running of the service and allow it to run on startup:

```bash
sudo nano /etc/systemd/system/cloudflared.service
```

```ini
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
```

Enable the `systemd` service to run on startup, then start the service and check its status:

```bash
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

#### Automatic way

<!-- markdownlint-disable code-block-style -->
!!! warning
    Keep in mind that this will install `cloudflared` as root.
<!-- markdownlint-enable code-block-style -->

Proceed to create a configuration file for `cloudflared` in `/etc/cloudflared` named `config.yml`:

```bash
sudo mkdir /etc/cloudflared/
sudo nano /etc/cloudflared/config.yml
```

Copy the following configuration:

```yaml
proxy-dns: true
proxy-dns-port: 5053
proxy-dns-upstream:
  - https://1.1.1.1/dns-query
  - https://1.0.0.1/dns-query
```

Now install the service via `cloudflared`'s [service command](https://developers.cloudflare.com/argo-tunnel/reference/arguments/#service-command):

```bash
sudo cloudflared service install
```

Start the `systemd` service and check its status:

```bash
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

Now test that it is working! Run the following `dig` command, a response should be returned similar to the one below:

```text
pi@raspberrypi:~ $ dig @127.0.0.1 -p 5053 google.com

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

### Configuring Pi-hole

Finally, configure Pi-hole to use the local `cloudflared` service as the upstream DNS server by specifying `127.0.0.1#5053` as the Custom DNS (IPv4):

![Screenshot of Pi-hole configuration](../images/DoHConfig.png)

(don't forget to hit Return or click on `Save`)

### Updating `cloudflared`

#### Manual way

```bash
wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-arm.tgz
tar -xvzf cloudflared-stable-linux-arm.tgz
sudo systemctl stop cloudflared
sudo cp ./cloudflared /usr/local/bin
sudo chmod +x /usr/local/bin/cloudflared
sudo systemctl start cloudflared
cloudflared -v
sudo systemctl status cloudflared
```

#### Automatic way

```bash
sudo cloudflared update
sudo systemctl restart cloudflared
```

### Uninstalling `cloudflared`

#### If installed the manual way

*Courtesy of <https://discourse.pi-hole.net/t/uninstall-cloudflare/21459/3>*

```bash
sudo systemctl stop cloudflared
sudo systemctl disable cloudflared
sudo systemctl daemon-reload
sudo deluser cloudflared
sudo rm /etc/default/cloudflared
sudo rm /etc/systemd/system/cloudflared.service
sudo rm /usr/local/bin/cloudflared
```

#### If installed with `cloudflare service install`

```bash
sudo cloudflared service uninstall
sudo systemctl daemon-reload
```

After the above, don't forget to change the DNS back to something else in Pi-hole's DNS settings!

[^guide]: Based on [this guide by Ben Dews | bendews.com](https://bendews.com/posts/implement-dns-over-https/)




