**FreeBSD jail and VNET introduction guide.**

We will be creating a FreeBSD jail using the [VNET(9)](https://www.freebsd.org/cgi/man.cgi?query=vnet&sektion=9) virtualized network stack, together with [EPAIR(4)](https://www.freebsd.org/cgi/man.cgi?query=epair&sektion=4&apropos=0&manpath=FreeBSD+12.1-RELEASE+and+Ports).

NOTE: This guide doesn't cover rootfs image and dataset creation.

The EPAIR is a pair of two virtual interfaces which are teoretically designed to work as a Ethernet crossover cable, thus linking them end to end. 

When using VNET with EPAIR, the epairXb interface is seen and threated as a physical interface by the Jail, and thus allowing us to do some cool stuff with our jail.

#### This confiruation will achieve the following setup ####
- a. Two logical interfaces, epairXa and epairXb, are created for each jail, where epairXa is assigned to the HOST and epairXb to the jail guest
- b. epairXb interfaces will be named jail0 inside each jail
- c. epairXa will be given an IP different from HOST's subnet upon jail's startup
- d. Jail will be reachable from the host through epairXa by assigning jail0 and IP on the same subnet of epairXa and by adding epairXa's IP as default gw for the jail

#### HOST configuration ####
> /etc/rc.conf 
```
ifconfig_vtnet0="inet 10.16.0.101 netmask 255.255.255.0"
gateway_enable="YES"

# Jail
jail_enable="YES"
jail_list=""
inetd_flags="-wW -a $EXT_IP"    # Restrict inetd to not interfere with jails

# PF
pf_enable="YES"
pf_rules="/etc/pf.conf"
pflog_enable="YES"
pflog_logfile="/var/log/pflog" 
pflog_flags=""                  # additional flags for pflogd startup
```
> /etc/pf.conf
```
ext_if="vtnet0"
IP_JAIL_www="192.168.69.254"
NET_JAIL="192.168.69.0/24"

scrub in all

# nat all jail traffic
nat on $ext_if inet from $NET_JAIL to any -> ($ext_if)

# Allow ICMP ping
pass inet proto icmp from any to any

# allowing all traffic IN/OUT (unsafe)
pass out
pass in
```
> /etc/jail.conf
```
# Global settings are applied to all jails

exec.system_user  = "root";
exec.jail_user    = "root";
mount.devfs;
allow.raw_sockets;
devfs_ruleset     = "5";

# Network, pre and post .exec rules
vnet;
vnet.interface    = "jail0";  # default vnet interface
exec.prestart    += "ifconfig $epair create up                 || echo 'Skipped creating epair (wtf?)'";
exec.created      = "ifconfig ${epair}b name jail0             || echo 'Skipped renaming ifdev to jail0'";
exec.clean;
exec.start        = "/bin/sh /etc/rc";
exec.stop         = "/bin/sh /etc/rc.shutdown";
exec.poststop    += "ifconfig jail0 -vnet $name"; # workaround to bug 238326
exec.poststop    += "ifconfig jail0 destroy"; # workaround creates jail0 on the host when stopping jail
services

# Per-jail settings
www {
    path          = "/jails/www0/12.1-RELEASE/root";
    host.hostname = "www";
    $epair        = "epair0";  # must be unique in every jail
    exec.poststart = "ifconfig epair0a 192.168.69.1 netmask 255.255.255.0";
    exec.consolelog = "/jails/www0/console.log";
}
```
#### Jail configuration ####
> /etc/rc.conf
```
ifconfig_jail0="inet 192.168.69.254 netmask 255.255.255.0"
defaultrouter="192.168.69.1"
```
#### Enabling services and starting our Jail ####
```
sudo sysctl net.inet.ip.forwarding=1 
sudo service pf start
sudo service pflog start
sudo service jail start www
 ```
#### The result ####
```
[admin@lockdown ~]$ jls
   JID  IP Address      Hostname                      Path
     1                  www                           /jails/www0/12.1-RELEASE/root
[admin@lockdown ~]$ 

[admin@lockdown ~]$ ifconfig epair0a 
epair0a: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=8<VLAN_MTU>
	ether 02:86:93:8f:00:0a
	inet6 fe80::86:93ff:fe8f:a%epair0a prefixlen 64 scopeid 0x4
	inet 192.168.69.1 netmask 0xffffff00 broadcast 192.168.69.255
	groups: epair
	media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
	status: active
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
[admin@lockdown ~]$

[admin@lockdown ~]$ netstat -nr4
Routing tables

Internet:
Destination        Gateway            Flags     Netif Expire
default            10.16.0.1          UGS      vtnet0
10.16.0.0/24       link#1             U        vtnet0
10.16.0.101        link#1             UHS         lo0
127.0.0.1          link#2             UH          lo0
192.168.69.0/24    link#4             U       epair0a
192.168.69.1       link#4             UHS         lo0
[admin@lockdown ~]$ 

[admin@lockdown ~]$ sudo jexec 1 ifconfig
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
	options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
	inet6 ::1 prefixlen 128
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1
	inet 127.0.0.1 netmask 0xff000000
	groups: lo
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
pflog0: flags=0<> metric 0 mtu 33160
	groups: pflog
jail0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=8<VLAN_MTU>
	ether 02:86:93:8f:00:0b
	inet 192.168.69.254 netmask 0xffffff00 broadcast 192.168.69.255
	groups: epair
	media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
	status: active
	nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
[admin@lockdown ~]$ 

[admin@lockdown ~]$ sudo jexec 1 ping -c2 google.com
PING google.com (172.217.169.46): 56 data bytes
64 bytes from 172.217.169.46: icmp_seq=0 ttl=55 time=6.920 ms
64 bytes from 172.217.169.46: icmp_seq=1 ttl=55 time=8.874 ms

--- google.com ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 6.920/7.897/8.874/0.977 ms
[admin@lockdown ~]$ 
```

Job done!


