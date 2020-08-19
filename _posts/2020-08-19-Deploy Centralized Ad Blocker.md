---
layout: post
title: "Deploy a centralized Ad Blocker in your network"
excerpt: "How to avoid installing ad blocker on all you devices"
excerpt_separator: "<!--more-->"
categories:
  - raspberry
tags:
  - linux
  - adblock
  - dns
last_modified_at: 2020-08-19T10:23:48-05:00
---

### It was DNS

DNS over HTTPS allows you to encrypt the traffic from your PC to the DNS server. Encrypting this channel prevents your ISP from seeing the sites you visit, which is why Mozilla has been labeled an Internet villain!


## DoH pro & Cons

1- Bypass censorship on DNS queries, in one click
2- Making it much more difficult for the ISP to maintain a navigation history
3- complicate Man in The Middle Attacks

Be careful however, enabling this feature has a big disadvantage if you do it directly on a network device or browser: parental filters will mostly be disabled since they only intercept requested domain names and prohibit them if they are in their blacklist. As the content is then encrypted, they will no longer be able to do so.

## Solution: Pi-Hole

The Pi-hole® is a [DNS sinkhole](https://en.wikipedia.org/wiki/DNS_sinkhole) that protects your devices from unwanted content, without installing any client-side software.

```sh
curl -sSL https://install.pi-hole.net | bash
```


