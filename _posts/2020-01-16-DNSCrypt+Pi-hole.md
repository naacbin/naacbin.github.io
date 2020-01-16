---
layout: single
title: How to increase your privacy with DNSCrypt and Pi-hole
date: 2020-01-16
excerpt: "I have always been aware of the lack of privacy on the Internet. But I only recently learned that anyone on your own personal network, or your ISP, could find out a lot about you through DNS queries, know which site you visit at what time, deduce the devices and services you use at home... Moreover, ISPs can censor websites, by blocking their name resolution. So, I will give you some insight on my setup to keep privacy and block known advertising and tracking domains."
classes: wide
mermaid: true
categories:
  - privacy
  - tutorial
tags:
  - linux
  - raspberry
  - dns
  - dnscrypt
  - pi-hole
  - docker
  - admin-sys
  - containers
---

<br>
I have always been aware of the lack of privacy on the Internet. But I only recently learned that anyone on your own personal network, or your ISP, could find out a lot about you through DNS queries, know which site you visit at what time, deduce the devices and services you use at home... Moreover, ISPs can censor websites, by blocking their name resolution.

Lately I've been hearing a lot about a solution to block _bad_ domain names, called Pi-hole, it's a DNS filtering solution. At first, I wanted to use it on my Synology NAS, but I ran into some problems and as I got a Raspberry Pi 4 2Go for Christmas, I decided to go for it.

So, I will give you some insight on my setup to keep privacy and block known advertising and tracking domains.
<br>

## Why DNS queries aren't secure ?

A DNS query is a question. In most cases a DNS request is used to convert a domain name like _example.com_ to an IP address like _127.0.0.1_, or the opposite. However this protocol isn't as secure as it should be. Indeed, all queries are sent as clear text, which means all the machines involved in the query chain know what you did and at what time. Moreover if your DNS server is corrupted, DNS queries can be modified. Below you can see examples of normal DNS queries vs. corrupt DNS queries.

**DNS query to a legit DNS server :**

<div class="mermaid">
  sequenceDiagram
    participant Client
    participant DNS server
    participant Gmail server
    Client->>DNS server: What is the IP of gmail.com ?
    DNS server->>Client: IP is X.X.X.X
    Client->>Gmail server: Here are my credentials for gmail
    Gmail server->>Client: Credential verified, you're logged in
</div>
<br>

**DNS query to a compromised DNS server :**

![corrupt-dns_diagram](/assets/images/dnscrypt_pi-hole/corrupt-dns_diagram.png)

This is a problem of integrity of the DNS server, to resolve it, **DNSSEC** has been created which fixes this issue but not the privacy ones. You can learn more about it here : [dnssec.vs.uni-due.de](https://dnssec.vs.uni-due.de/) and check if your DNS uses it (I recommend avoiding DNS that do not use DNSSEC). To resolve privacy issues you can use **DoH** (DNS-over-HTTPS) which encrypts your traffic using HTTPS so ISP can't see your traffic, only the DNS server which receives the request can. To do that I will use **DNSCrypt-proxy**. In addition I will add a **DNSCrypt-server** on my VPS, however, it's not a mandatory step.
<br>

## Raspberry Pi

### Stuff

I'm using the default 16Go SD card that came with my Raspberry Pi, an Ethernet cable (but you can use Wi-Fi), and a USB-C power supply (5V/3A).
<br>

### Install

Follow this [guide](https://hackernoon.com/raspberry-pi-headless-install-462ccabd75d0) to have a proper installation of the latest Raspbian version without a GUI. Don't forget to download the **Lite version**.
<br>

### Setup

Connect through SSH and run raspi-config.

```bash
ssh pi@raspberry.pi # Can work if you can't find IP
```

```bash
sudo raspi-config # Use this command when you are connected to configure your Raspberry
```

![raspi-config](/assets/images/dnscrypt_pi-hole/raspi-config.png)

- Change your password.
- Change localization and keyboard layout if needed.

Update your system and remove unnecessary package :

```bash
sudo -i # Spawns a new shell as the root user
apt-get update && apt-get dist-upgrade -y
apt-get autoremove
```

Setup a static IP :

```bash
vi dhcpcd.conf # If you're not familiar with vi use nano instead
```

![static-ip](/assets/images/dnscrypt_pi-hole/static-ip.png)

Uncomment the two lines under **_Example static IP configuration_**. The first line corresponds to the interface (run `ip a` on linux to check your current IP configuration, interfaces...), and the second one to the private IP you want to attribute, choose one that is not currently used in your network.

<br>
## Pi-hole

### What is Pi-hole ?

Pi-hole is an **open source** DNS server, which acts as a [DNS sinkhole](https://en.wikipedia.org/wiki/DNS_sinkhole) to block advertisements and Web trackers. It can act as a DHCP server too. Pi-hole receives DNS queries from clients connected to it and requests a response from a DNS resolver if the domain name isn't in its blocklist/blacklist.

Pi-hole has a default blocklist which contains about 110,000 domains to block. However, you need to find other sources of blocklists to have a really complete one, or you could customize it to make your own.

**Be careful, blocking some domain can break access to a website, so you need to use whitelists.**

![pi-hole_network_diagram"](/assets/images/dnscrypt_pi-hole/pi-hole_network_diagram.png)

Image from [virtualthoughts.blob.core.windows.net](https://virtualthoughts.blob.core.windows.net).
<br>

### Install

```bash
curl -sSL https://install.pi-hole.net | bash
```

When it asks you to choose upstream DNS, choose DNSWatch or Quad9 (we will change it later). Always check the default configuration for next steps.
<br>

### Setup

Set a new password for the administration of Pi-hole :

```bash
pihole -a -p
```

Once it's done, go to http://X.X.X.X/admin (where X.X.X.X corresponds to the IP of the raspberry) and connect to the interface.

![pi-hole_interface](/assets/images/dnscrypt_pi-hole/pi-hole_interface.png)

Go to Settings -> Blocklists and add all those links, it will block about 1,6M domains.

```
https://dbl.oisd.nl/
https://blocklist.site/app/dl/crypto
https://blocklist.site/app/dl/drugs
https://blocklist.site/app/dl/fraud
https://blocklist.site/app/dl/fakenews
https://blocklist.site/app/dl/gambling
https://blocklist.site/app/dl/malware
https://blocklist.site/app/dl/phishing
https://blocklist.site/app/dl/piracy
https://blocklist.site/app/dl/ransomware
https://blocklist.site/app/dl/scam
https://blocklist.site/app/dl/spam
https://raw.githubusercontent.com/deathbybandaid/piholeparser/master/Subscribable-Lists/ParsedBlacklists/Block-EU-Cookie-Shit-List.txt
https://gist.githubusercontent.com/unknownFalleN/3f38e2daa8a98caff1b0d965c2b89b25/raw/53035c634a699817f3ab0800102b267253912114/xiaomi_dns_block.lst
```

The first weeks you will probably get some errors trying to access on some sites, review query logs to check if the problem comes from Pi-hole. If that's the case, just whitelist the domain and check if that fixes the issue.

**Be aware :**
Pi-hole can easily be bypassed by setting another IP address into DNS configuration, on smartphones, laptops... Furthermore some devices like IoT (_Internet of Things_) or TVs have hard-coded DNS servers, so it will bypass Pi-hole too.

When all of this is done, check that everything works fine when accessing a website.

<br>
## DNSCrypt-proxy

### What is DNSCrypt-proxy ?

> DNSCrypt is a protocol that authenticates communications between a DNS client and a DNS resolver.

DNSCrypt-proxy is an implementation of DNSCrypt written in Golang by [jedisct1](https://github.com/jedisct1) which support DoH and Anonymized-DNSCrypt too.
<br>

### Install

- Check [repository](https://github.com/jedisct1/DNSCrypt-proxy/releases/) for the latest version, download it with the following command, replace `X.X.XX` with the version number you find.
  ```bash
  wget https://github.com/jedisct1/dnscrypt-proxy/releases/download/X.X.XX/dnscrypt-proxy-linux_arm-X.X.XX.tar.gz
  ```
- Extract from tar.
  ```bash
  tar xzvf dnscrypt-proxy-linux_arm-X.X.XX.tar.gz
  ```
- Rename the extracted folder.
  ```bash
  mv linux-arm dnscrypt-proxy && cd dnscrypt-proxy
  ```
- Create a configuration file based on the example one.
  ```bash
  cp example-dnscrypt-proxy.toml dnscrypt-proxy.toml
  ```
<br>

### Setup

Edit the file.

```bash
vi dnscrypt-proxy.toml
```

- Add the server you want to use in `server_names`, if you are located in Belgium, Germany, France or Switzerland I recommend you use `opennic-R4SAS` by typing the following line : `server_names = ['opennic-R4SAS']`

- Edit the port, since `53` is already being used by Pi-hole, use 5300 for example.
  `listen_addresses = ['127.0.0.1:5300']`

- Set :

  - `ipv6_servers = false`
  - `dnscrypt_servers = true`
  - `doh_servers = true`
  - `require_dnssec = true`
  - `require_nolog = true`
  - `require_nofilter = true`
  - `cache = false` in the _DNS Cache_ part

* We are now going to use [Anonymized DNS](https://github.com/DNSCrypt/DNSCrypt-proxy/wiki/Anonymized-DNS), it helps anonymizing queries by sending encrypted requests to a relay that can't decrypt them, and forward them to a DNSCrypt server who can decrypt them but don't know your IP address. To do so, you need to configure the routes option at the end of the file. You can see a list of relays [here](https://github.com/DNSCrypt/DNSCrypt-resolvers/blob/master/v2/relays.md). Do not use a relay from the same organization as the DNSCrypt server because it would give the same information to the server you are trying to reach in 2 differents steps instead of one.

**DNS query using Anonymized DNS :**

<div class="mermaid">
  sequenceDiagram
    participant Client
    participant DNSCrypt relay
    participant DNSCrypt server
    Client-->>DNSCrypt relay: Encrypted Request
    DNSCrypt relay->>DNSCrypt server: Encrypted Request
    Note over DNSCrypt server: Decrypt request and <br> send response
    DNSCrypt server-->>DNSCrypt relay: Encrypted Response
    DNSCrypt relay->>Client: Encrypted Response
    Note over Client: Decrypt response 
</div>

- Install DNSCrypt-proxy service.

  ```bash
  ./dnscrypt-proxy -service install
  ```

- Start the new service.

  ```bash
  ./dnscrypt-proxy -service start
  ```
<br>

### Other setup :

On the web interface for Pi-hole, go to Settings -> DNS. Check the same boxes as in the screenshot below, and change 5300 to the port you chose in `dnscrypt-proxy.toml`:

![upstream_dns](/assets/images/dnscrypt_pi-hole/upstream_dns.png)

Enable DNSSEC validation :

```bash
echo "proxy-dnssec" >> /etc/dnsmasq.d/02-DNSCrypt.conf
```

Startup activation :

```bash
systemctl enable dnscrypt-proxy.service
systemctl restart dnscrypt-proxy.service
systemctl enable pihole-FTL.service
systemctl restart pihole-FTL.service
```

Check if you got an IP address from the DNSCrypt server with the following command :

```bash
./dnscrypt-proxy -resolve framasoft.org
```

To check DNSSEC, you can go to [dnssec.vs.uni-due.de](https://dnssec.vs.uni-due.de/).
To check DNSLeak, navigate to [dnsleaktest.com](https://www.dnsleaktest.com/) and choose _Extented test_.

<br>
## DNSCrypt-server

### What is DNSCrypt-server ?

DNSCrypt-server is a DNS server that provides DNSSEC, DoH and a caching DNS resolver by default. It can be used to escape censorship and keep your life more private.
<br>

### Install

If you haven't installed docker yet, follow this [guide](https://docs.docker.com/install/linux/docker-ce/debian/). When you're done, you can download the following docker container.

```bash
docker pull jedisct1/dnscrypt-server
```

Create a directory and its parents if needed to store the data from the docker container.

```bash
mkdir -p /etc/dnscrypt-server/keys
```

Run the container with the following command :

```bash
docker run --name=dnscrypt-server -p 443:443/udp -p 443:443/tcp --net=host --ulimit nofile=90000:90000 --restart=unless-stopped -v /etc/dnscrypt-server/keys:/opt/encrypted-dns/etc/keys jedisct1/dnscrypt-server init -N example.com  -E 'X.X.X.X:443'
```

- `docker run` is the command used to launch Docker containers.
- `--name` gives a name to your container.
- `-p` binds a port from inside the container to your host machine. The first part is the local port of your machine, the second part is the local port of your container, the last part sets the communication protocol communication, either TCP or UDP. In our case we use both.
- `--ulimit nofile=90000:90000` increases the number of open files in the container to 90000 (by default it is set to 1020).
- `--restart=unless-stopped` restarts the container only when any user executes a command to stop the container, not when it fails because of an error.
- `--net=host` is for using host network instead of the default bridge.
- `--volume`, or `-v` in our case, is used to map a host volume to a docker volume. The first is the host volume, the second is the docker volume.
<br>

The following arguments are specific to the DNSCrypt-server container:

- `-N` is the name of our DNSCrypt-server, choose whatever you want.
- `-E` is the IP address to which the client will connect.
- `X.X.X.X` represents the public IP of your VPS.

<br>
To see if our container is running check :

```bash
docker ps
```

If it is not the case use :

```bash
docker start dnscrypt-server
```

NOTE: to remove the container, use

```bash
docker rm --force dnscrypt-server
```
See more documentation on [github](https://github.com/DNSCrypt/DNSCrypt-server-docker).

<br>
<br>

I would like to thanks **@HexPandaa** who read over my article to add readability and corrected my mistakes, you can go and have a look at his [github](https://github.com/HexPandaa).

Thank you very much for reading my first article, I hope this one interested you. If it is the case don't hesitate to follow me on my twitter.

