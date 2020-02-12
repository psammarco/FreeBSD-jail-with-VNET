**FreeBSD jail and VNET introduction guide.**

We will be creating a FreeBSD jail using the [VNET(9)](https://www.freebsd.org/cgi/man.cgi?query=vnet&sektion=9) virtualized network stack, together with [EPAIR(4)](https://www.freebsd.org/cgi/man.cgi?query=epair&sektion=4&apropos=0&manpath=FreeBSD+12.1-RELEASE+and+Ports).

The EPAIR is a pair of two virtual interfaces which are teoretically designed to work as a Ethernet crossover cable, thus linking them end to end. 

When using VNET with EPAIR, the epairXb interface is seen and threated as a physical interface by the Jail, and thus allowing us to do some cool stuff with our jail.

This confiruation will achieve the following setup:
#### a. Two logical interfaces, epairXa and epairXb, are created for each jail, where epairXa is assigned to the HOST and epairXb to the jail guest; ####
#### b. epairXb interfaces will be named jail0 inside each jail;####
#### c. epairXa will be given an IP different from HOST's subnet upon jail's startup;####
#### d. Jail will be reachable from the host through epairXa by assigning jail0 and IP on the same subnet of epairXa and by adding epairXa's IP as default gw for the jail;####
