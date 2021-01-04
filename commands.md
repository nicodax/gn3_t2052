# DHCP configuration

create configuration file on the DHCP server : 

```
/ # vi /etc/dhcp/dhcpd.conf
```

copy the content of /dhcpd/dhcpd.conf in /etc/dhcp/dhcd.conf. The commented line ```option domain-name-servers 192.168.0.1;``` will be usefull later on

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

At the moment, only the client knows ther is a resolver server in the subnet. edit the config file of all other server hosts and uncomment the line ```up echo nameserver 192.168.0.1 > /etc/resolv.conf``` in the config files

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

in order to allow the whole subnet to access the formation.lab zone, the resolver needs to know he can forward requests concerning that zone. On the resolver, uncomment the ```zone "0.168.192.in-addr.arpa.lab" { ... };``` block in the section ```# CUSTOM ZONES SECTION``` of the /etc/named/named.conf file

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

# Setting up an apache web server

open a terminal outside of gns3

```
cd /home/user/Images/apache
```

compile the apache image

```
docker build . -t me/myapache
```

in gns3, delete the www host and replace it by a new End Device > Admin - Apache. edit config file as needed

right_click > configure. select the Advanced section. in "Additional directories to make persistent [etc]" box, add ```/etc/bind/```. Make sure its config file is correct and run the web server :

```
/run-httpd.sh &
```

# Configure Apache

configure a server name. edit the httpd.config file and look for the line ```ServerName www.example.com```, uncomment it and change it to ```ServerName www.formation.lab```

create an index.html file in the Web Documents root directory :

```
vi /var/www/html/index.html
```

copy the content of /Admin-Apache-1/www/html/index.html in /var/www/html/index.html

restart the server

access http://www.formation.lab from the client's web browser. the index.html file should load

### configuration exercices

the configuration file is in /etc/httpd/conf/httpd.conf

* change the web document root to /home/user/web :
	* create the directories user and web to make the afore mentioned pathname valid
	* move the index.html file from /var/www/html to /home/user/web
	* edit the dhcpd.conf file. look for the line ```DocumentRoot /var/www/html``` and change it to ```DocumentRoot /home/user/web```. a few lines below should be a line ```<Directory "/var/www/html">```. change the pathname there as well
	* restart the server
	* access www.formation.lab from the client's web browser. the index.html file should load

```
mkdir -p /home/user/web
mv /var/www/html/index.html /home/user/web/index.html
```

* change the admin email : edit the dhcpd.conf file. look for the line ```ServerAdmin root@localhost``` and change the email.
* change the default html file name for each directory to test.html :
	* edit the dhcpd.conf file. look for the line ```DirectoryIndex index.html``` and change it to ```DirectoryIndex test.html```
	* change the index.html in /home/user/web to test.html
	* resart the server
	* access www.formation.lab from the client's web browser. the test.html file should load

```
mv /home/user/web/index.html /home/user/web/test.html
```

* the server should not listen to ip addresses outside of the local network
	* edit the dhcpd.conf file. look for the block defined by ```<Directory "/var/www/html">```. inside, change the line ```Require all granted``` by ```Require ip 192.168.0.0/24```
	* restart the server
	* access www.formation.lab from the client's web browser. the test.html file should load

* the server listens to ports 80 and 8080
	* edit the dhcpd.conf file. look for the line ```Listen 80```. below, add the line ```Listen 8080```

* the access to /home/user/web/site1/perso is protected by authentification
	* create a .htpasswd file in /etc/httpd/conf/
	* edit the dhcpd.conf file. create a new ```<Directory>``` block and configure it to ask for authentification

```
htpasswd -c /etc/httpd/conf/.htpasswd user
```

enter the password for user

```
user123
```

```
<Directory "/home/user/web/site1/perso">
    AuthType Basic
    AuthName "Restricted Content"
    AuthUserFile /etc/apache2/.htpasswd
    Require valid-user
</Directory>
```

# Setting up an SMTP mail server

open a terminal outside of gns3

```
cd /home/user/Images/mailServers
```

compile the apache image

```
docker build . -t me/mailserver
```

in gns3, delete the www host and replace it by a new End Device > Admin - Mail Server. edit config file as needed

right_click > configure. select the Advanced section. in "Additional directories to make persistent [etc]" box, add ```/etc/mail/```. Make sure its config file is correct and run the web server :

```
/usr/sbin/sendmail -bd -q15s -v &
```

# SMTP server configuration (with POP/IMAP)

create two linux users on mail server

```
sh-4.4# useradd bob
sh-4.4# passwd bob
Changing password for user bob.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
sh-4.4# 
sh-4.4# useradd amy
sh-4.4# passwd amy
Changing password for user amy.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

```mail``` command examples :

```
# send mail to bob (from root)
sh-4.4# mail bob
Subject: hi bob
it's me, root
EOT
[1]+  Done                    /usr/sbin/sendmail -bd -q15s -v
```

```
# check bob's mailbox
sh-4.4# mail -f bob
Heirloom Mail version 12.5 7/5/10.  Type ? for help.
"bob": 5 messages
>   1 root                  Mon Jan  4 16:53  21/774   
    2 root                  Mon Jan  4 16:57  22/767   "Test Subject"
    3 root                  Mon Jan  4 17:05  22/693   "test"
    4 root                  Mon Jan  4 17:10  22/691   "hi bob"
    5 root                  Mon Jan  4 17:16  22/774   "localhost"
& 4
Message  4:
From root@mail  Mon Jan  4 17:10:44 2021
Return-Path: <root@mail>
From: root <root@mail>
Date: Mon, 04 Jan 2021 17:09:44 +0000
To: bob@mail
Subject: hi bob
User-Agent: Heirloom mailx 12.5 7/5/10
Content-Type: text/plain; charset=us-ascii
Status: RO

it's me

```