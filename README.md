# Bypassing VPN with dnsmasq+chinadns

## Motivation
A VPN client ususally sends/receives all network packages through the remote server. 
Though the behaviour is required by private networking, 
we may need to bypass the traffic of the client in some situations. For example,

* Some services are only accessable in the subnet of the client.
* The public bandwidth of a VPN server is limited, and making all connection private is unneccessary.

One solution is the [chnroutes.py](https://github.com/jimmyxu/chnroutes) script.
It crawls data from APNIC and generates routing rules for local connections.
The script works great, but it is inefficient when visiting websites with CDN.
Specifically, when resolving a domain name,
the client could forward the query to the VPN server, and get a result from a DNS in the remote subnet
(/etc/resolv.conf contains both the vpn server and default DNSs of the local subnet).
If a website deploys CDN and has a content server in the remote subnet, 
the DNS may return an ip which can not be applied with chnroutes.py's routing rules.
As a result, the following data traffic will be forwarded to the VPN server (then to the content server).
On the other hand, using DNSs of the local subnet may not be a good idea (e.g., DNS poisoning). 
Thus, besides bypassing the networking of data, we also need to bypassing DNS queries. 

```
             firewall
                ++
                ||
+--------+      ||       +--------+
|        |      ||       |        |
|   ○    |      ||       |   ●    |
| client+----------------->server |
|        |      ||       |        |
|        |      ||       |        |
+--------+      ||       +--------+
  client        ||         server
  subnet        ++         subnet




   □                          ■
content      firewall      content
server1         ++         server2
                ||
                ||            ^
                ||            |
   +--------------------------+
   |            ||
   v            ||
   ○ +----------------------> ●
 client         ||          server
                ||   return   +
                ||   ■'s ip   |
                ||            |
                ||            v
   ☆           ++            ★
  DNS1                       DNS2
```

We describe a configuration which bypasses different DNS queries to different 
(local/remote subnet) DNSs. The key tool is [chinadns](https://github.com/shadowsocks/ChinaDNS), 
which has done almost 95% of the job. We accomplish the remaining 5% with the help of dnsmasq and a proper setting
of /etc/resolv.conf.

## Environment
* A VPN server.
* A linux client (Arch Linux) on which the VPN connection can be established.

## Overview
Diagram of Local DNS


## Configurations

### chinadns

### dnsmasq

### /etc/resolv.conf

## Reference

