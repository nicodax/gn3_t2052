# DHCP configuration

create configuration file on the DHCP server : 

```
/ # vi /etc/dhcp/dhcpd.conf
```

copy the content of /dhcpd/dhcpd.conf in /etc/dhcp/dhcd.conf. The line ```option domain-name-servers 192.168.0.1;``` will be usefull later on, as will the other commented lines

run the configuration file to start the DHCP server :

```
/ # dhcpd -cf /etc/dhcp/dhcpd.conf
```

launch a Wireshark capture on the client's connection :
* initiate a new IP lease : ```/gns3/bin/busybox udhcpc -x hostname:client```
* filter Wireshark's traces with ```udp.port == 68```. There should be four results : DHCP Discover, DHCP Offer, DHCP Request and DHCP ACK
* close the Wireshark capture

Check if the client has received an IP address :

```
/ # ifconfig eth0
```

Check the client's connectivity with ping :

```
/ # ping 8.8.8.8
```

# DNS resolver

create configuration file on the resolver server : 

```
/ # vi /etc/bind/named.conf
```

copy the content of /resolver/named.conf in /etc/bind/named.conf. The section ```# CUSTOM ZONES SECTION``` will be usefull later on, as will some others

run the configuration file to start the bind resolver server :

```
/ # named -g &
```

make sure the resolver works properly. From the soa server :

```
/ # dig @192.168.0.1 www.google.com
```

if it works properly, the answer section should contain the IPv4 address associated with www.google.com

```
/ # dig @192.168.0.1 www.google.com

; <<>> DiG 9.14.12 <<>> @192.168.0.1 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40516
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: d06400bfefcf6d2b3dfd4b4d5ff09f03905dd24db1e27a0d (good)
;; QUESTION SECTION:
;www.google.com.			IN	A

;; ANSWER SECTION:
www.google.com.		300	IN	A	216.58.208.100

;; Query time: 884 msec
;; SERVER: 192.168.0.1#53(192.168.0.1)
;; WHEN: Sat Jan 02 16:27:47 UTC 2021
;; MSG SIZE  rcvd: 87
```

Uncomment the line ```option domain-name-servers 192.168.0.1;``` in the DHCP server configuration file. The DHCP server will now announce the resolver in the subnet

The client should now be able to ping google.com :

```
/ # ping www.google.com
```

# Configure DNS zone

All server hosts should be configured with a static IP. Their configuration files (accessible with right_click > edit config) should look like this :

```
#
# This is a sample network config uncomment lines to configure the network
#


# Static config for eth0
auto eth0
iface eth0 inet static
	address 192.168.0.1
	netmask 255.255.255.0
	gateway 192.168.0.254
#	up echo nameserver 192.168.0.1 > /etc/resolv.conf

# DHCP config for eth0
# auto eth0
# iface eth0 inet dhcp
```

if everything looks good, create configuration file on the soa server : 

```
/ # vi /etc/bind/named.conf
```

copy the content of /soa/named.conf in /etc/bind/named.conf. The section ```# ZONES SECTION``` will be usefull later on

## configure forward zone

create the forward zone file on the soa server :

```
/ # vi /etc/bind/db.formation.lab
```

copy the content of /soa/db.formation.lab in /etc/bind/formation.lab

check both configurations.
* use named-checkconf to check the named.conf file

```
/ # named-checkconf /etc/bind/named.conf
```

* use named-checkzone to check the db.formation.lab file

```
/ # named-checkzone formation.lab /etc/bind/db.formation.lab
```

In order to allow the whole subnet to access the formation.lab zone, the resolver needs to know he can forward requests concerning that zone. On the resolver, uncomment the ```zone "formation.lab" { ... };``` block in the section ```# CUSTOM ZONES SECTION``` of the /etc/named/named.conf file

At the moment, only the client knows ther is a resolver server in the subnet. edit the config file of all other server hosts and uncomment the line ```up echo nameserver 192.168.0.1 > /etc/resolv.conf```

The server hosts won't know about this until you restart them. In order to save the configuration files and zone files, save the path to their respective directories :
* on the resolver and soa servers, right_click > configure. select the Advanced section. in "Additional directories to make persistent [etc]" box, add ```/etc/bind/```
* on the dhcp server, in that same box, add ```/etc/dhcp/```

Stop all nodes in GNS3 and wait for them to close. Restart all nodes in GNS3 and wait for them to boot

run the DHCP server configuration file and the resolver and soa server bind configuration files :

```
/ # dhcpd -cf /etc/dhcp/dhcpd.conf          # DHCP server
/ # named -g &                              # resolver server
/ # named -c /etc/bind/named.conf -g &      # soa server
```

from different machines, try pinging other machines' names. for example, with the client :

```
/ # ping www.formation.lab
/ # ping ns.formation.lab
/ # ping resolv.formation.lab
```

## configure reverse zone

create the reverse zone file on the soa server :

```
/ # vi /etc/bind/db.0.168.192.in-addr.arpa
```

copy the content of /soa/db.0.168.192.in-addr.arpa in /etc/bind/db.0.168.192.in-addr.arpa

check both configurations.
* use named-checkconf to check the named.conf file

```
/ # named-checkconf /etc/bind/named.conf
```

* use named-checkzone to check the db.0.168.192.in-addr.arpa file

```
/ # named-checkzone 0.168.192.in-addr.arpa /etc/bind/db.0.168.192.in-addr.arpa
```

in order to allow the whole subnet to access the formation.lab zone, the resolver needs to know he can forward requests concerning that zone. On the resolver, uncomment the zone "0.168.192.in-addr.arpa.lab" { ... }; block in the section # CUSTOM ZONES SECTION of the /etc/named/named.conf file

restart both the resolver and the soa

run the resolver and soa server bind configuration files :

```
/ # named -g &                              # resolver server
/ # named -c /etc/bind/named.conf -g &      # soa server
```

check connectivity with nslookup :

```
/ # nslookup 192.168.0.1
Server:		192.168.0.1
Address:	192.168.0.1:53

Non-authoritative answer:
1.0.168.192.in-addr.arpa	name = resolv.formation.lab
```