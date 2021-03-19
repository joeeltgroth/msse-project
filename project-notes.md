


### First attempt at networking computers on a subnet via a separate switch and bridging via WiFi
Worked on setting up local network.  Following this: 
https://wiki.dd-wrt.com/wiki/index.php/Linking_Subnets_with_Static_Routes
Had a big struggle with this.  Though I was able to setup a 10.0.0.0 network, I wasn't able to get routing (or bridging?) to work.  I tried having my stand-alone 10.0.0.0 network, but tried to make one of the boxes WiFi wireless interfaces into a bridge, but got errors back perhaps indicating that wasn't allowed... not sure why.  Since my project is more about Kubernetes than networking, and the networking I care about is AWS, GCP, Azure, vSphere, etc, not home-style Linksys hardware, I decided to leave that behind... 


### Second attempt at networking using base 192.168.1.0 network on existing home network
Just allocate 192.168.1.x static addresses to the boxes by connecting an unmanaged swith into the main router and get on with the show of installing kubernetes. 


### The resulting network
![Physical network](/images/network.png)

Have been using this doc to understand how to configure networks on Ubuntu: 
https://ubuntu.com/server/docs/network-configuration


### My process for setting up a Latitude E5540 with Ubuntu linux

First, I downloaded [Ubuntu Server 20.10](https://ubuntu.com/download/server).  I got an .iso file.  
I used commands like these to burn a CVD disk becuase I couldn't get a flash drive to boot on the old hardware I have. 
```bash
hdiutil convert -format UDRW -o ubuntu-20.10-live-server-amd64.img ubuntu-20.10-live-server-amd64.iso 
diskutil list
diskutil umount /dev/disk2s1
sudo dd if=./ubuntu-20.10-live-server-amd64.img of=/dev/rdisk2 bs=1m
```
The above commands resulted in a bootable DVD. 

The reason I choose Ubuntu Server is that I wanted a lightweight server with no graphical user interface.  I looked into 
Photon OS Minimal, but got stuck and given that I use Ubuntu more often at work it seemed like the logical choice. 

To install Ubuntu, I started the Latitude E5540 and pressed F12 until the boot menu came up, and I choose
CD/DVD as the boot medium. I installed the Ubuntu server fully on the disk (these computers were prior wiped clean with dban.org's boot/nuke product). 

On most of the boxes, I let them come up using DHCP, then later changed their IP address to 
a static. 

I also installed an ssh server on each box. This allows me to connect from my MacBook, the computer
I normally use, and not have to go to the command line on each computer's individual keyboard. 

[Pictures of the installation process](installing-ubuntu.md)


### Setting IP Addresses
A tool called "netplan" is used on Ubuntu Linux (probably other distributions also), to configure 
networking on linux. 

The important file is /etc/netplan/00-installer-config.yaml. (note that the name can vary a bit, but it is the only file in that directory).  Below I am configuring with 
dhcp turned off, and 192.168.1.50 as the address. The gateway is the address of my 
Linksys router/switch. 

Below I am showing the nameservers, but I couldn't configure that section until I'd setup
a bind9 DNS server (which was then listening to DNS queries on 192.168.1.50)
```
network:
  ethernets:
    enp10s0:
      dhcp4: no
      addresses: [192.168.1.50/24]
      gateway4: 192.168.1.1
      nameservers:
        search: [joesmsse.com]
        addresses: [192.168.1.50, 192.168.1.1]
  version: 2
```
Useful Commands when setting the IP:  

`ip a` - show the network interfaces

`sudo lshw -class network` - get detailed network interface details. 

`sudo ethtool eth4` - changes ethernet card settings (auto-negotiation, port speed... ) 

`vi /etc/netplan/00-installer-config.yaml` - edit the netplan configuration to set static ip, gateway, and nameservers. 

`sudo netplan apply` - apply changes to the netplan

`cat /etc/os-release` - what version of Linux you are running


### Installing a DNS Server. 

The instructions I use later for Kubernetes don't require a DNS server, but I wanted to set one up to see how it is done and learn more about how DNS is configured. 

I setup the box I called f6thw as with bind9 as a DNS server while referencing the [Ubuntu DNS documentation](https://ubuntu.com/server/docs/service-domain-name-service-dns)

1. Install bind9: `sudo apt install bind9`
2. Upgrade all the packages (not strictly necessary, but good practice): `sudo apt dist-upgrade -y` (See [How to List Available Updates...](https://www.poftut.com/how-to-list-available-updates-and-updateable-packages-with-apt-apt-get-aptitude-commands/) 
for apt command explanations. 
3. Edit the bind9 configuration file `vi /etc/bind/named.conf.options`. 
5. Restat the bind9 service: `sudo systemctl restart bind9.service`. 
6. dig www.htch.com - note the query time is significantly less after the 1st time. 

When doing a dig, remember to look for the answer section.  For example, in the query below, the question section is just echoing back what I asked for, but the ANSWER section shows the address I want.  If the answer wasn't found, there won't be an answer section. 
```
# dig haproxy.joesmsse.com

; <<>> DiG 9.16.6-Ubuntu <<>> haproxy.joesmsse.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53797
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;haproxy.joesmsse.com.		IN	A

;; ANSWER SECTION:
haproxy.joesmsse.com.	7145	IN	CNAME	ns.joesmsse.com.
ns.joesmsse.com.	7145	IN	A	192.168.1.50

;; Query time: 0 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Mon Mar 15 20:05:49 UTC 2021
;; MSG SIZE  rcvd: 82
```

It was interesting for me to see the named.conf.local file points to two other "db" files: one that
specifies the mapping from name to address, and the other from address to name. 

The bind9 configuration. Key thing here is the zone and reverse zone definition and the path to the files that have the mappings. 
```
joe@f6thw:~$ cat /etc/bind/named.conf.local

zone "joesmsse.com" {
    type master;
    file "/etc/bind/db.joesmsse.com";
};

zone "1.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/db.192";
};
```

The mapping from name to IP. 
I think this is saying it serves everyting in joesmsse.com, and that the nameserver "ns" can be found on 192.168.1.50.  And, I wanted a name "haproxy" (cause this box is also where I plan to install that), so that is set to the same address (CNAME) that ns is.  Pretty sure this is where we'll set all the mappings for all the static k8s cluster boxes in the network. 
```
joe@f6thw:~$ cat /etc/bind/db.joesmsse.com
;
; BIND data file for joesmsse
;
$TTL	604800
@	IN	SOA	joesmsse.com. root.joesmsse.com. (
			      6		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	joesmsse.com.
@	IN	A	192.168.1.50
@	IN	AAAA	::1
ns      IN      A       192.168.1.50
haproxy IN      CNAME   ns
f6thw   IN      CNAME   ns
ctrl1   IN      A       192.168.1.51
ctrl2   IN      A       192.168.1.52
frhn    IN      CNAME   ctrl1
worker1 IN      A       192.168.1.53
worker2 IN      A       192.168.1.54
worker3 IN      A       192.168.1.55
```

The mapping from ip to name.  Not 100% sure if this is necessary, but does allow a reverse nslookup as shown. 
```
joe@f6thw:~$ cat /etc/bind/db.192
;
; BIND reverse data file for local loopback interface
;
$TTL	604800
@	IN	SOA	ns.joesmsse.com. root.joesmsse.com. (
			      5		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	ns.
50	IN	PTR	ns.joesmsse.com.
51      IN      PTR     ctrl1.joesmsse.com.
52      IN      PTR     ctrl2.joesmsse.com.
53      IN      PTR     worker1.joesmsse.com.
54      IN      PTR     worker2.joesmsse.com.
55      IN      PTR     worker3.joesmsse.com.

# nslookup 192.168.1.50
50.1.168.192.in-addr.arpa	name = ns.joesmsse.com.

```

Useful commands when installing DNS: 

`vi /etc/bind/named.conf.local` - edit the bind9 configuration

`named-checkzone joesmsse.com /etc/bind/db.joesmsse.com` validate your zone file

`named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192` validate your reverse zone file. 

`sudo systemctl restart bind9.service` - restart the bind9 DNS server 

`systemd-resolve --status` show you what DNS server you are resolving against. 

`vi /etc/netplan/00-installer-config.yaml` edit your static ip, gateway and resolver

`dig haproxy.joesmsse.com` query the DNS server 


### Configuring HAProxy as a Load Balancer

I did [Step 1: Configuring HAProxy as a Load Balancer](https://www.inap.com/blog/deploying-kubernetes-on-bare-metal/) but the `nc -v $KBLB_ADDR 6443` says "... Connection refused". So, I've either mis-configured HAProxy, or I need to have the Kubernetes control plane running on the ctl1 and ctlr2 boxes before haproxy can serve any traffic to them. 

More to come... 

