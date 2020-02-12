**FreeBSD jail and VNET introduction guide.**

We will be creating a FreeBSD jail using the [VNET(9)](https://www.freebsd.org/cgi/man.cgi?query=vnet&sektion=9) virtualized network stack, together with [EPAIR(4)](https://www.freebsd.org/cgi/man.cgi?query=epair&sektion=4&apropos=0&manpath=FreeBSD+12.1-RELEASE+and+Ports).

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
 
 
 
