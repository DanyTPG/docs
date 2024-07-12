---
title: Network Address Translation (NAT) and iptables
date: 2024-07-12 20:00:00 +0330
categories: [Linux, iptables]
tags: [iptables, nat]     # TAG names should always be lowercase
---

This post will only include key points and examples from Karl Rupp's great article on NAT[^1].

## NAT Table Chains
There are 3 main chains in the NAT table found in iptables: **PREROUTING**, **OUTPUT** and **POSTROUTING**

The `PREROUTING` and `POSTROUTING` chains are the most important ones and as their name implies `PREROUTING` is responsible for handling the packets that arrive at the network interface and `POSTROUTING` for the ones that leave it.

## Activate required features

Before we start packet manipulations we have to enable some required features.

> IMPORTANT: Activate IP-forwarding in the kernel which is disabled by default!
{: .prompt-warning }

```shell
# Activate ipv4 forwarding
echo "1" > /proc/sys/net/ipv4/ip_forward
   
# Load various modules. Usually they are already loaded 
# (especially for newer kernels), in that case 
# the following commands are not needed.
    
# Load iptables module:
modprobe ip_tables
   
# activate connection tracking
# (connection's status are taken into account)
modprobe ip_conntrack
```

## iptables

An iptables command has the following pattern:

```shell
# Abstract structure of an iptables instruction:
iptables [-t table] command [match pattern] [action]
```

### Choosing a table
For NAT we always choose the NAT table. 
```shell
# Choosing the nat-table
# (further arguments abbreviated by [...])
iptables -t nat [...]
```

### Commands
The most important commands are the following:
```shell
# In the following "chain" represents
# one of the chains PREROUTING, OUTPUT and POSTROUTING

# add a rule:
iptables -t nat -A chain [...]

# list rules:
iptables -t nat -L

# remove user-defined chain with index 'myindex':
iptables -t nat -D chain myindex

# Remove all rules in chain 'chain':

iptables -t nat -F chain
```

### Choosing match patterns
To manipulate specific packets we need to use match patterns. Some of the popular ones include:
```shell
# actions to be taken on matched packets
# will be abbreviated by '[...]'.
# Depending on the match pattern the appropriate chain is selected.

# TCP packets from 192.168.1.2:
iptables -t nat -A POSTROUTING -p tcp -s 192.168.1.2 [...]

# UDP packets to 192.168.1.2:
iptables -t nat -A POSTROUTING -p udp -d 192.168.1.2 [...]

# all packets from 192.168.x.x arriving at eth0:
iptables -t nat -A PREROUTING -s 192.168.0.0/16 -i eth0 [...]

# all packets except TCP packets and except packets from 192.168.1.2:
iptables -t nat -A PREROUTING -p ! tcp -s ! 192.168.1.2 [...]

# packets leaving at eth1:
iptables -t nat -A POSTROUTING -o eth1 [...]

# TCP packets from 192.168.1.2, port 12345 to 12356
# to 123.123.123.123, Port 22
# (a backslash indicates contination at the next line)
iptables -t nat -A POSTROUTING -p tcp -s 192.168.1.2 \
   --sport 12345:12356 -d 123.123.123.123 --dport 22 [...]
```
For most of the switches there exists a long form, e.g. --source instead of -s. Using them makes the whole instruction longer but more readable, especially if you are new to iptables.

### Actions for matched patterns
Now that we have specified the packets we want, we need to choose the actions that will be applied to them. The actions for the NAT table include **SNAT**, **MASQUERADE**, **DNAT**, and **REDIRECT**, all of which will be preceded by `-j` to become meaningful.

#### Source-NAT (SNAT) - Change sender statically
For SNAT we explicitly specify the new Source-IP of the packet. Since SNAT is only meaningful for packets which will be leaving the network interface, it is only used within the **POSTROUTING** chain.
```shell
# Options for SNAT (abstract of manual page)
--to-source <ipaddr>[-<ipaddr>][:port-port] 
```

#### MASQUERADE - Change the sender to router's IP-Address
Using MASQUERADE will change the source address of every packet to the IP of the router's outgoing interface. The advantage of MASQUERADE over SNAT is that if we have have a dynamic IP address from the internet provider, we wouldn't have to change the rule every time our public IP changes. However if we have a static IP address, SNAT is the best choice because it is faster than MASQUERADE which has to check the current IP address of the outgoing network interface at every packet.
Unlike SNAT, MASQUERADE does not provide any further options.
```shell
# Connect a LAN to the internet
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
``` 

#### Destination-NAT (DNAT) - Changing the receipient
If we want to change the destination address IP of a packet, we use DNAT. This feature is usually used when our destination is behind a NAT. An example of this would be **port forwarding** for reaching clients in our home network which are on a private network behind our router.
```shell
# Options for DNAT (abstract of manual page)
--to-destination <ipaddr>[-<ipaddr>][:port-port] 
```

#### REDIRECT - Redirect packets to local machine
A special case of DNAT is REDIRECT. Packets are redirected to a local port of the router, enabling for example transparent proxying. As for DNAT, REDIRECT acts within the PREROUTING and the OUTPUT chain respectively.
```shell
# Options for REDIRECT (abstract of manual page)
--to-ports <port>[-<port>] 
```

## Applications

In this section we will take a look at some use cases.

### Transparent Proxying
```shell
# (local net at eth0, proxy server at port 8080)
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 \
   -j REDIRECT --to-ports 8080 
```
### Redirect SSH from port 5000 to port 22
```shell
iptables -t nat -A PREROUTING -p tcp --dport 5000 -j REDIRECT --to-ports 22
```
### redirect and masquerade
```shell
# redirect port 5001 to port 110 (POP3) at 123.123.123.123:
iptables -t nat -A PREROUTING -p tcp --dport 5001 \
       -j DNAT --to-destination 123.123.123.123:110

# Change sender to redirecting machine:
iptables -t nat -A POSTROUTING -p tcp --dport 110 \
   -j MASQUERADE
```
### redirect http-Traffic going to Port 80 to 111.111.111.111:5002
```shell
iptables -t nat -A OUTPUT -p tcp --dport 80 \
       -j DNAT --to-destination 111.111.111.111:5002
```
### running a server behind a NAT router
Let us assume that we have an HTTP server with IP `192.168.1.2` and our router has the IP address `192.168.1.1` and is connected to the internet over its second network interface with IP `1.1.1.1`. To reach the HTTP server from outside, type
```shell
# redirect http traffic to 192.168.1.2:
iptables -t nat -A PREROUTING -p tcp -i eth1 --dport 80 -j DNAT --to 192.168.1.2
```
We will now be able to access the HTTP server from outside using the public IP `1.1.1.1`.

[^1]: [NAT with Linux and iptables](https://www.karlrupp.net/en/computer/nat_tutorial)
