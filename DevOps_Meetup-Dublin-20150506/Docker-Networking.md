footer: © Vicente De Luca, 2015 - http://www.nethero.org - @zenvdeluca
slidenumbers: true

#[FIT]Docker
#[FIT]Container Linking
#for production
#[FIT]*From a NetOps dude view*


![right](docker-net.jpg)

---
![right](itguy.jpg)

#Vicente De Luca
*About me*
- Network Engineer @ Zendesk
- jack of all trades
- hacking since 90's
- IPv6 Addicted and Evangelist

http://nethero.org
http://github.com/zenvdeluca
vdeluca@zendesk.com
@zenvdeluca

---

![right](smoke_puffs.jpg)
# Docker engine 
*Network support*
- Bridge (default)
- Port mapping (NAT)
- Host mapping (--net host)
- Container (--net container)
- No network (--net none)

---
![right](lbridge.jpg)
# Bridge mode 
*Characteristics:*
- Docker default
- Sequential IP assignment
- Linux Worker = gateway :worried:

---

![right](small-open-door.jpg)
# Port mapping (NAT) 
*Characteristics:*
- Add operations overhead :worried:
- Linux Worker = gateway :worried:
- Add difficulty for logging :worried:
- Resource exhaustion risks :worried:

**Don't scale!**
- at 15 containers =~ 2000 ports
- Avoid NAT444
- Docker NAT != CGN

---

![right](share.jpg)
# Container namespace 
*Characteristics:*
- share other container net namespace
- recent feature

---

![right](overload.jpg)

# Host mapping 
*Characteristics:*
- Bind at worker net namespace :neutral_face:
- Mostly one IP, all containers :worried: 
- No network ns isolation :worried:
- Concurrent apps: some risks :worried:

#[FIT] A good start, but...

---

#[FIT] as you grow...
![original](containers.jpg)

---

#[FIT] bad things can happen
![](grow.jpg)

---

![right filtered](issues.jpg)
# Common issues

 - Bind TCP conflicts at same port
   *Non stop code deployment?*
 <br> 
 - Port exhaustion risk when:
   *TCP conns to same dst_ip:port*
   *>= ~500 new req/s*
   *default TIME_WAIT=60s*

---

![right filtered](tshoot.gif)
# Complicate Troubleshooting
  
 - netflow
   - all apps run at worker IP
 <br>
 - i.e: Which app is starving?
   - easy for inbound
   - hard for outbound

---

![right filtered](workaround.jpg)
# DOOOEEET RIGHT
## FROM THE BEGINNING!

---

# Proposed approach
*One IP per container:*
- containers <=> containers, no NAT
- nodes <=> containers, no NAT
- IP that sees itself as, is the same others see it
- != containers *CAN* expose same service port 
- Cluster managers requirement, like Kubernetes
- Supports IPv4 (DHCP) and IPv6 (SLAAC)  

---
![](iot-ip.jpg)
# One IP per container
# Different tracks

---

![right filtered](rockets.jpg)
# Dynamic Routing
### Building complicated solutions to simple problems

- Advertise running container IPs to network routers
- Uses BGP protocol
  - NetOps OK, DevOps not so

www.projectcalico.com
<br>

---

![right](internet.jpg)
# Overlay Network
### For Cloud Providers or Staging:###
*Flannel / Weaver*
 - overhead increase
 - reduces the effective MTU size
 - may impact TTL
 - tunnel = lower performance 
 - reduces network visibility

---

![](shutup.jpg)

#Stop chit chat, please?

---

# Simple is Beautiful
*DHCP:*
- reliable (RFC2131)
- everybody knows how it works
<br>
*Bridge:*
- Part of kernel since middle ages 
- Replaceable by fancy vSwitches
- Network gears does L3 router :+1:

![right](kiss.jpg) 

---

# Network Architecture

![inline fit](arch.jpg)

---

# One IP per container

*Ingredients:*
- DHCP Server providing dynamic leases
- Bridge interface (br0) with eth0 attached
- One DHCP client per "RPM package on steroids"
- DHCP processes running at worker userspace 
- A Network wrapper will do the magic

---

# Network wrapper - #!/bin/bash
####My Simplified fork from github.com/jpetazzo/pipework 
-generate unique locally MAC
-Creates a new network namespace
-Associate the new veth with the our br0 interface
-Run and maintain the DHCP client
-Garbage collection for orphan DHCP processes

---

# MAC Address Generation Algorithm
*Generate a locally unique MAC:*
*XX:IP:IP:IP:YY:ZZ*

- XX = restricted random least significant bits = 10
- IP:IP:IP = worker IPv4 3 last octets in HEX
- YY:ZZ = random portion

---

```bash
root@docker1 vdeluca]# network-wrapper
Syntax:
network-wrapper inspect[-all] <guestID>
network-wrapper <HostOS-interface> [-i Container-interface] <guestID> dhcp [macaddr]
 
[root@docker1 vdeluca]# network-wrapper br0 $(docker run --net none -d nginx) dhcp
{
    "Container": "4b94d0da2047",
        "NetworkSettings": {
          "HWAddr": "c6:64:50:63:3b:86",
          "Bridge": "br0",
          "IPAddress": "10.100.80.190",
          "IPPrefixLen": "24",
          "Gateway": "10.100.80.1",
          "DNS": "10.100.64.9, 10.100.66.9",
          "DHCP_PID": "23384"
} }
[root@docker1 vdeluca]# curl 4b94d0da2047.stage.lab.zdsys.com
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

---

![](brain.jpg)
#[FIT] QUICK LIVE DEMO

---

#TL;DR
- no one architecture fits all needs
- currently everything is a trade off, 
- but reliability / scaling is non-negotiable core value
- Learned (often the hard way) that significant floats in network architectures are not easy problems to solve.

---

#More info about
- Tricking Docker daemon to provide IP per container
- Auto scaling load balancing using consul-template

check it out at my blog [http://nethero.org](http://nethero.org) 
(or invite me for the next meetup :smile:)

---

![FIT](codeship.jpg)

---

![](chat.jpg)
#[FIT] OPEN CHAT
#[FIT] &
#[FIT] QUESTIONS

---

![original](bar.jpg)
#[FIT]THANK YOU!
#[FIT] 2nd round starts now at the Barge 
#[FIT] just across the street! 
#[FIT]Sorry about our streaming fellows :(


