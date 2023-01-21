# Secure Network Name Resolution (DNS) via systemd-resolved

Suppose you want several things to be true for a computer:
1. use a specific DNS server for every interface/location (WIFI or VPN),
2. use DNSSEC, and
3. use TLS.

There are a few ways to do this, but my favorite is to use systemd-resolved. This is what I will 
demonstrate here.

Configure ```/etc/systemd/resolved.conf``` ...

```sudo nano -wl /etc/systemd/resolved.conf```

Edit this following lines...
```
DNS=[2606:4700:4700::1002]:853#security.cloudflare-dns.com [2606:4700:4700::1112]:853#security.cloudflare-dns.com 1.1.1.2:853#security.cloudflare-dns.com 1.0.0.2:853#security.cloudflare-dns.com
Domains=<YOUR_DOMAIN_HERE>
DNSSEC=yes
DNSOverTLS=yes
Cache=yes
DNSStubListener=yes
```

You may notice that I used the Cloudflare for Families malware addresses. If you are not 
familiar, this is a free service that attempts to not serve addresses for domain that are 
*known* to serve malware. This should **NOT** be your primary defense against bad people on the 
Internet-of-Mean-People, but it is a good layer in an overall security strategy.   

Now, restart systemd-resolved.
```sudo systemctl restart systemd-resolved.service```

You should now see two resolved listening on two UDP and two TCP ports:

```
>> sudo ss -ltup | grep resolv
udp   UNCONN 0      0              127.0.0.54:domain      0.0.0.0:*    users:(("systemd-resolve",pid=1143,fd=15))            
udp   UNCONN 0      0           127.0.0.53%lo:domain      0.0.0.0:*    users:(("systemd-resolve",pid=1143,fd=13))            
tcp   LISTEN 0      4096        127.0.0.53%lo:domain      0.0.0.0:*    users:(("systemd-resolve",pid=1143,fd=14))            
tcp   LISTEN 0      4096           127.0.0.54:domain      0.0.0.0:*    users:(("systemd-resolve",pid=1143,fd=16))            
```

You should also see programs that read ```cat /etc/resolv.conf``` directly are sent to 
systemd-resolved for name resolution.

```
>> cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
search YOUR_DOMAIN_HERE
```

Now, we need to do one more thing. NetworkManager may still append DNS servers provided by a 
DHCP server. When you go to your favorite bakery and start your laptop, do you want to use their 
DNS servers even if it is to simply find to your VPN server(s)? Probably not. To disable that, 
we need to do to things: 
1. find your device ID, and
2. tell NetworkManager to ignore DNS servers from DHCP.

Find your device id as follows:
```
>> sudo nmcli device status
DEVICE  TYPE      STATE        CONNECTION 
enp5s0  ethernet  connected    enp5s0     
enp6s0  ethernet  connected    enp6s0     
wlp4s0  wifi      unavailable  --         
lo      loopback  unmanaged    --   
```

Here, you can see I have three interfaces: two ethernet (enpXs0) and one Wi-Fi (wlp4s0). To 
disable DNS from DHCP, do the following except replace enp5s0 with the ID for your device.

```
sudo nmcli connection modify enp5s0 ipv4.ignore-auto-dns yes
sudo nmcli connection modify enp5s0 ipv6.ignore-auto-dns yes
```

Finally, we need to restart the device to ensure the changes are truly made.

```
sudo nmcli device disconnect enp5s0
sudo nmcli device connect enp5s0
```

Awesome, now all DNS queries will be handled (and cached) by one service that uses your 
preferred DNS servers, DNSSEC, and TLS whether using Wi-Fi or VNP. 

If you want to further tighten your security, I always disable mDNS and LLMNR (both use 
broadcasting to find devices on your local network), because using them in any public network 
increases your attack surface. To disable these, do the following.

```
>> sudo nano -wl /etc/systemd/resolved.conf
MulticastDNS=no
LLMNR=no

>> sudo nmcli connection modify enp5s0 connection.mdns no
>> sudo nmcli connection modify enp5s0 connection.llmnr no
```
