# Bypassing VPN with dnsmasq+chinadns

## Overview
A VPN client ususally sends/receives all network packages through the remote server. 
Though the behaviour is required by private networking, 
we may need to bypass the traffic of the client in some situations. For example,

* Some services are only accessable in the subnet of the client.
* The public bandwidth of a VPN server is limited, and making all connection private is unneccessary.

One solution is the [chnroutes.py](https://github.com/jimmyxu/chnroutes) script.
It crawls data from APNIC and generates routing rules for local connections.
The script works great, but it is inefficient when visiting websites with CDN (Figure 1.a, 1.b).
Specifically, when resolving a domain name,
the client could forward the query to the VPN server, and get a result from a DNS in the remote subnet
(`/etc/resolv.conf` contains both the vpn server and default DNSs of the local subnet).
If a website deploys CDN and has a content server in the remote subnet, 
the DNS may return an ip which can not be applied with chnroutes.py's routing rules.
As a result, the following data traffic will be forwarded to the VPN server (then to the content server).
On the other hand, using DNSs of the local subnet may not be a good idea (e.g., DNS poisoning). 
Thus, besides bypassing the networking of data, we also need to bypass DNS queries. 

```
                             ■           □                                    step2:
                          content     content                                 compare returned
    □                     server2     server1                  ■              results with a
 content                     ^           ^                  content           predefined ip list    firewall
 server1    firewall         |           |                  server2                                  +---+
             +---+           |           |         firewall                       +---+              |---|
             |---|           +           v          +---+                 ○ +---> | ? | +-----------------> ●
    ○  +-------------------> ●           ○          |---|      ●        client    +---+     step1:   |---|server
  client     |---|         server      client       |---|    server              chinadns   query    +---+  +
             +---+  return   +           +  return  |---|                           +       DNS1&2          |
                    ■'s ip   |           |  □'s ip  +---+                           |                       |
    ☆                       |           |                     ★                   |                       |
   DNS1                      v           v                    DNS2                  v                       v
                             ★          ☆                                         ☆                      ★
                            DNS2        DNS1                                       DNS1                    DNS2


(a) chnroutes.py fails to obtain       (b) the preferred traffic.              (c) Bypassing with chinadns.
    the ip of content server1.

                                               Figure 1
```

We describe a configuration which bypasses different DNS queries to different 
(local/remote subnet) DNSs. The key tool is [chinadns](https://github.com/shadowsocks/ChinaDNS), 
which has done 95% of the job (Figure 1.c). We accomplish the remaining 5% with the help of `dnsmasq` and a proper setting
of `/etc/resolv.conf`. Figure 2 shows the overall setting.

```
        +---------------+     +-------+      +--------+      VPN        DNSs of the
 +--->  |127.0.0.1      | +-> |dnsmasq| +--> |chinadns| +--> gateway +->VPN server
 query  +---------------+     +-------+      +----+---+ |
        |               |                         |     |
        |VPN gateway    |                         |     +--> DNS1
        |               |                         |     |
        |DNS1, DNS2,... |                         |     |
        +---------------+                         |     +--> DNS2
        /etc/resolv.conf  +-----------------------+

                                      Figure 2
```

## Environment
* A linux client (Arch Linux) with a successful VPN connection.

## Configurations

### chinadns
Install `chinadns`. For Arch Linux, an [AUR package](https://aur.archlinux.org/packages/chinadns/) is available.
Check the installation with
```
systemctl start chinadns
systemctl status chinadns
```
It will setup a local DNS server at 127.0.0.1 with port 5353 
(chrome may occupy ports 5353, see instructions in AUR or official repo for a different port).

### dnsmasq
Install `dnsmasq`, and edit `/etc/dnsmasq.conf`
```
no-resolv
no-poll
server=127.0.0.1#5353
```
* `no-resolv`: `dnsmasq` will not read nameservers in `/etc/resolv.conf`
* `no-poll`: `dnsmasq` will not poll nameservers in `/etc/resolv.conf`
* `server=127.0.0.1#5353`: set 127.0.0.1:5353 as an upstream DNS server, which is established by `chinadns`.  
`dnsmasq` will establish another local DNS with ordinary port 53.

### /etc/resolv.conf
To involve `chinadns` in the process of name resolving, we need to send all DNS queries to the local dnsmasq server 127.0.0.1, then to `chinadns` (rather than the VPN gateway or other DNS servers).
Prepend (create if it's not there) `/etc/resolv.conf.head` with
```
nameserver=127.0.0.1
```
The content of `/etc/resolv.conf.head` will be added at the top of `/etc/resolv.conf`

### chnroutes.py
Adding routing rules generated by chnroutes.py (see the document of chnroutes.py). 

## Reference
\[1\]https://github.com/shadowsocks/ChinaDNS

\[2\]https://github.com/jimmyxu/chnroutes
