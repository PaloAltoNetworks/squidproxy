# Squid Proxy patch to run tproxy preserve the 5 tuple of client traffic
By default squid proxy changes the source port number for client traffic while configured to handle connections in tproxy mode. This patch will make squid not change the source port number so that it can work with the decryption broker feature. This readme also has pointers to configure squid proxy as ICAP client to a DLP (in this case clamav via c-icap and squidclamav plugin for c-icap). 

More information about how to configure Decryption Broker Feature can be found here at https://docs.paloaltonetworks.com/pan-os/8-1/pan-os-admin/decryption/decryption-broker

## Installation of required packages: 
There are two packages required for installing squid with tproxy patch:

Build tools like gcc compiler and make etc.

libcap devel rpm

Here is how to install these on centos and ubuntu:

### CentOS:

yum groupinstall "Development Tools"

yum install libcap-devel

### Ubuntu:

sudo apt-get update

sudo apt-get install build-essential

sudo apt-get install libcap-dev

## Downloading the source code and applying the patch:

The patch should work with squid version 4.1 to 4.9

curl -o squid-4.1.tar.gz http://www.squid-cache.org/Versions/v4/squid-4.1.tar.gz

tar xzvf squid-4.1.tar.gz

cd squid-4.1

patch -p1 < ~/squid.patch

## Compilation: 

Configure with a prefix for installation folder and check for warnings in config.log to make sure tproxy feature has no warning: 

./configure --prefix=/opt/squid/ --enable-linux-tproxy --enable-linux-netfilter

grep WARNING config.log



#Check that there are no warnings for tproxy

make -j12

make install



cd /opt/squid/

#By default squid uses "nobody" linux user for running

chown -R nobody:nobody var



## Configuration: 

Please follow this guide for more details:

https://wiki.squid-cache.org/Features/Tproxy4


Here is a script which can be run at the bootup time to configure the tproxy settings: (Note: Some of these settings can be added to linux config files to make them permanent)

### Routing for outgoing traffic 

#this can go to /etc/rc.local

ip -f inet rule del fwmark 1 lookup 100

ip -f inet route del local default dev lo table 100

ip -f inet rule add fwmark 1 lookup 100

ip -f inet route add local default dev lo table 100

#this can go to /etc/sysctl.conf

sysctl -w net.ipv4.ip_forward=1

sysctl -w net.ipv4.conf.default.rp_filter=0

sysctl -w net.ipv4.conf.all.rp_filter=0

### IPTABLE rules

iptables -t mangle -N DIVERT

iptables -t mangle -A PREROUTING -p tcp -m socket -j DIVERT

iptables -t mangle -A DIVERT -j MARK --set-mark 1

iptables -t mangle -A DIVERT -j ACCEPT

iptables -t mangle -A PREROUTING -p tcp --dport 443 -j TPROXY --tproxy-mark 0x1/0x1 --on-port 3129

iptables -I INPUT -j ACCEPT

### Selinux setting 
#This can be made persistent with setsebool -P boolean-name

setsebool squid_connect_any=1

setsebool squid_use_tproxy=1



## Squid settings in squid.conf:

### ACL:

Add all the client subnets in the acl so that squid can accept the connections from those subnets.

cd /opt/squid/etc

vi squid.conf

add client subnets to the acl localnet. Config format is acl <aclname> src <ip range/subnet>

### TPROXY Port:

Add the port 3129 for tproxy:

http_port w.x.y.z:3129 tproxy # Where w.x.y.z is the broker chain first ip and assigned on squid proxy machine


### ICAP setting:

Here is a sample configuration which works with c-icap and clamav DLP. Please check following url for more configuration options https://wiki.squid-cache.org/Features/ICAP 


For clamav please check following link (its easier to install on Ubuntu) 

http://squidclamav.darold.net/documentation.html#documentation


icap_enable on

adaptation_send_client_ip on

icap_client_username_header X-Authenticated-User

icap_service service_req reqmod_precache bypass=1 icap://192.168.101.1:1344/squidclamav

adaptation_access service_req allow all

icap_service service_resp respmod_precache bypass=1 icap://192.168.101.1:1344/squidclamav

adaptation_access service_resp allow all



### Routing configuration:

There should be 4 interfaces on the squid proxy machine:

* Management interface

* Interface to connect to Icap server

* Broker chain IN interface

* Broker chain OUT interface

### Routes:

The default route of the squid machine should point to the Broker out interface on the machine so that all the client traffic going to internet is sent back to firewall.

There need to be a route for connecting to management interface from local clients for ssh

A route need to be present for the DLP Icap server, so that squid can connect to ICAP server

### Starting squidproxy

squid can be configured as service, here is how to run squid manually

#### To run in foreground:

sudo /usr/local/squid/sbin/squid -N

#### To run in background:

sudo /usr/local/squid/sbin/squid

### Stopping squidproxy:

sudo /usr/local/squid/sbin/squid -k shutdown

once started logs can be seen in /opt/squid/var/log/

## Other notes:

IP in the subnet 100.64.0.0/10 may not work with tproxy

It is recommended to disable the Ubnutu/Centos advanced firewall service delete all iptable rules then configure squid iptable rules with an allow all in INPUT table. Once tproxy is working then add IP table rules manually to avoid any conflicts.

If it comes to debugging tproxy then following kernel logs may be enabled and seen via dmesg (Reference: https://www.kernel.org/doc/html/v4.19/admin-guide/dynamic-debug-howto.html) This option requires kernel to be compiled with dynamic debugging enabled. 

sudo su

cd /sys/kernel/debug

grep tproxy dynamic-debug/control

for each file listed execute following command:

echo 'filepath/filename:line-number +p' > dynamic-debug/control

for e.g.

echo 'net/netfilter/xt_TPROXY.c:186 +p' > dynamic-debug/control

send some traffic and check debugs with command "sudo dmesg"

_/* Feel free to provide me feedback to improve this documentation via opening an issue. */_
