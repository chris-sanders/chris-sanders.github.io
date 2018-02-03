---
layout: post
title:  "MAAS for the home"
subtitle: "The unsupported configuration"
date: 2018-02-02 20:00:00 +0000
gh-repo: chris-sanders/maas-lxd
gh-badge: [star, fork, follow]
image: "/img/maas-logo.svg"
tags: [homelab, maas, lxd]
---

MAAS is designed to run in a data center where it expects to have control of DNS
and DHCP. The use of an external DHCP server is listed as 'may work but not
supported' in the MAAS documentation. This guide will describe how I configured
MAAS to coexist with a home router with the router providing DHCP and DNS. Some
of this will depend on the router allowing advanced configurations and not all
routers will support it.

There are some limitations when MAAS doesn't control DHCP and DNS but none of
those limitations affect me in home lab use today. One day I might switch my DNS
and DHCP from the consumer grade Asus I have today to a more robust solution.
That's a project for another day and another post, the configurations I describe
here are a good stop-gap until that time.

If you already have a MAAS server you can skip directly to the 
[Network Configuration](#network-configuration) section.

## Installing MAAS

I've chosen to install MAAS in a LXD container. While this requires a minimal
amount of extra setup it allows me to migrate the LXD to another machine in the
future and keeps MAAS separated from anything else on the machine. I have a
repository ([maas-lxd][maas-lxd]) with some scripts and profiles to help set it
up. You can use the buttons at the top of this post to star or fork the
repository to make it easier to find later. 

### MAAS-LXD

The README file from the [maas-lxd][maas-lxd] repository explains how to use it
but is intended to setup a self contained environment where LXD is providing
NAT (the LXD default). For the home lab we're going to modify that to use a
bridged setup since we want MAAS to receive PXE requests from the network.

You'll need to have LXD installed, here's how you can do that with snaps.
```bash
$ sudo snap install lxd
```
On my host machine ```br0``` is a bridge I already have configured in
```/etc/network/interface```. Be sure you have a brdige ready for use, here's my
configuration as an example.
```
auto enp4s0
iface enp4s0 inet manual

auto br0
iface br0 inet dhcp
  bridge-ifaces enp4s0
  bridge-ports enp4s0
  up ifconfig enp4s0 up

```
We want to reconfigure LXD to use this bridge so the containers will be bridged
onto the network. To configure LXD to use ```br0``` instead of LXD's
default ```lxdbr0```. Use the configuration wizard.
```bash
$ sudo lxd init
```
LXD configuration isn't the focus of this post, but if you want to learn more
check it out on [github](https://github.com/lxc/lxd/blob/master/doc/index.md).

Clone the maas-lxd repository:
```bash
$ git clone https://github.com/chris-sanders/maas-lxd.git
$ cd maas-lxd
```
Edit the ```maas-profile```, at the bottom modify the devices by removing the
eth1 device and setting eth0 to use ```br0```. 
```diff
devices:
  eth0:
    name: eth0
    nictype: bridged
-   parent: lxdbr0                              
+   parent: br0                              
    type: nic                                   
- eth1:                                         
-   name: eth1                                  
-   nictype: bridged                            
-   parent: lxdbr0                              
-   type: nic
```
Also at the top of the profile modify the device configuration to match.
```diff
config:
  - type: physical
    name: eth0
    subnets:
      - type: dhcp
- - type: physical                          
-   name: eth1                              
  - type: bridge
    name: br0
    bridge_interfaces:
-     - eth1
+     - eth0 
    subnets:
      - type: dhcp
```
We're ready to create the container, run the make-maas.sh script with a
single argument which is the name you want your container to have. I'm using the
container name ``maas-test`` for this setup.
```bash
$ ./make-maas.sh maas-test
Creating container maas-test
Creating maas-test
Starting maas-test
Sleeping to wait for IP
LXC Stable branch not scripted for dnsmasq settings PXE will not work
To install the backport: apt install -t xenial-backports lxc lxc-client
MAAS will become available at: http://192.168.1.56/MAAS with user/password admin/admin
```
I'm doing this on Xenial, if you're running a newer LXD you won't see the error
about 'Stable branch' shown above. It won't affect anything we aren't using the
dnsmasq setting of LXD in this setup.

You can see the new container with lxd's list command.
```bash
$ lxc list
+---------------+---------+----------------------+------+------------+-----------+
|     NAME      |  STATE  |         IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+---------------+---------+----------------------+------+------------+-----------+
| maas-test     | RUNNING | 192.168.1.56 (eth0)  |      | PERSISTENT | 0         |
+---------------+---------+----------------------+------+------------+-----------+

```
The container is running but it has a lot to install and configure.  Cloud-init
is installing packages for maas, bridge utils, qemu/kvm, and configuring a
network for qemu. You can watch the progress in the container by tailing syslog
from inside the container.
```bash
$ lxc exec maas-test -- tail -f /var/log/syslog
```
Once it has completed check that the ```br0``` interface was added on the container.

```bash
$ lxc list
+---------------+---------+----------------------+------+------------+-----------+
|     NAME      |  STATE  |         IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+---------------+---------+----------------------+------+------------+-----------+
| maas-test     | RUNNING | 192.168.1.56 (eth0)  |      | PERSISTENT | 0         |
|               |         | 192.168.1.56 (br0)   |      |            |           |
+---------------+---------+----------------------+------+------------+-----------+

```
Having both interfaces dhcp will wreck havok on the containers networking, so
remove the dhcp from ```eth0``` and let ```br0``` manage it.

```bash
$ lxc exec maas-test -- vim /etc/network/interfaces.d/50-cloud-init.cfg
```
Remove the line shown below.
```diff
auto lo
iface lo inet loopback

auto eth0
-iface eth0 inet dhcp

auto br0
iface br0 inet dhcp
    bridge_ports eth0
```
Reboot the container
```bash
$ lxc restart maas-test
```
Now the container should have a single ```br0``` interface.
```bash
$ lxc list
+---------------+---------+----------------------+------+------------+-----------+
|     NAME      |  STATE  |         IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+---------------+---------+----------------------+------+------------+-----------+
| maas-test     | RUNNING | 192.168.1.56 (br0)   |      | PERSISTENT | 0         |
+---------------+---------+----------------------+------+------------+-----------+

```
At this point you can log in at the URL that the make-maas script provided
earlier.
```
MAAS will become available at: http://192.168.1.56/MAAS with user/password admin/admin
```
You should provide a static IP address for your MAAS container. If the IP
address changes you will have to update configurations in maas and your
router. There is a script in the scripts folder ```set_url.sh``` which will
update the IP address in MAAS if you want to do that now and set it to something
differnt from what it got on dhcp.

## Network Configuration

In order for MAAS to commission a node and provide the operating system the
machine must PXE boot from MAAS. This means the router that is providing DHCP
will need to forward PXE requests to MAAS. Many consumer routers today will
allow this as an advanced setting.

Containers created on MAAS managed machines _do not_ DHCP. Instead they get a
static IP assigned by MAAS. We will have to account for that when setting aside
IP space, also those machines will not register with the routers DNS. MAAS
deployed machines will show up in both DNS servers and MAAS deployed containers
will only be in the MAAS DNS Server since they do not DHCP. Not a problem, just
something to be aware of.

For IP space I'm be using.

| IP / Range  | Used For |
| ----------  | -------- |
| 192.168.1.1 | Router   |
| 192.168.0.5 | MAAS (static)|
| 192.168.1.2   - 192.168.1.254 | DHCP Range |
| 192.168.0.1   - 192.168.0.99 | Static Router assignments |
| 192.168.0.100 - 192.168.0.254 | Static MAAS assignments |

### Router configuration (ASUS RT-AC88U)

#### PXE redirect

To setup PXE requests to be redirected to MAAS you have to first enable SSH
access to the router. The setting is found in
```Administration/System/Services/Enable SSH```. Set it to "LAN only". Once set
you can ssh into your router with the user/pass you use to log into the web UI.
```bash
$ ssh admin@192.168.1.1
```
Once you're on the router you want to edit the file /etc/dnsmasq.conf
```bash
admin@gateway:/tmp/home/root# vi /etc/dnsmasq.conf
```
Add the following line to the bottom, notice I'm using the MAAS address.
```
dhcp-boot=pxelinux.0,,192.168.0.5
```
Now the router will provide DHCP and DNS but redirect PXE to MAAS.

#### Subnetmask

Since we're using IP space outside of the standard 192.168.1.x range that home
routers frequently come configured with, the router needs to be configured to
route the extra addresses in the 192.168.0.x range by adjusting the subnet
mask from ```255.255.255.0``` to ```255.255.254.0```.  

This setting is found on the settings page ```LAN/LAN IP```

![subnet mask](/img/maas/router-subnetmask.png){:.img-shadow .img-rounded}

#### DNS Server

Add the MAAS IP as an additional DNS server, and be sure "Advertise router's IP"
is enabled. This will put MAAS first in the DNS list and the router second. If
the MAAS machine is down for some reason DNS will work the same as before. 

This setting is found on the settings page ```LAN/DHCP Server```

![DNS Settings](/img/maas/router-dns.png)

### MAAS Configuration

#### DNS Forwarding

When logging into MAAS for the first time set DNS forwarding to forward to the
router.

![DNS forward](/img/maas/dns-forwarder.png)

If your MAAS is already running set it in ```Settings```

![DNS forward](/img/maas/maas-settings-dns.png)

#### Configure Subnet

In MAAS go to the tab ```Subnets``` and in the drop down box on the right
labeled "Add" choose "Space". Fill in the a name and click the green "Add space"
button to save it. I used "home-space" for the name.

![add space](/img/maas/add-space.png)

Now select the Subnet ```192.168.0.0/23``` from the table below. Verify that
your CIDR, Gateway, and DNS are set correctly. These are the settings that
match the IP table shown above and the subnetmask setup on the router.
Additionally, add your space to the Space field.

![subnet summary](/img/maas/subnet-settings.png)

Finally scroll down to the Reserved section and add reservations via the
"Reserve range" button. Set reservations which cover the DHCP and Static
assignments from the table above.

![reservations](/img/maas/reserved-ranges.png)

With this configuration MAAS will use IP address in the CIDR, which are not
reserved, for assignemnt to LXD containers. MAAS and the local router are
now configured so you can use MAAS without affecting the pre-maas network.
DHCP/DNS from the router are still used and if MAAS is down it won't affect your
non-maas network clients.

## Testing MAAS PXE without DHCP

Time to verify everything works with a KVM inside the MAAS LXD. That will
confirm that MAAS is working as expected and the KVM can be used as the
controller node for juju.

virsh-install is available in the container to create the VM but the --pxe
option only applies to the first boot. A quick work around is to generate the
VM, export the xml, destory the domain, and create it from the xml. At least you
don't have to write the xml file!

Run this command to get the KVM setup.
```bash
$ lxc exec maas-test -- virt-install \
--name=juju-controller \
--os-type=Linux \
--os-variant=ubuntu16.04 \
--ram=4096 \
--vcpus=2 \
--disk size=10 \
--pxe \
--network bridge:br0 \
--graphics vnc,listen=127.0.0.1 --noautoconsole &&\
lxc exec maas-test -- virsh dumpxml juju-controller > controller.xml &&\
lxc exec maas-test -- virsh destroy juju-controller &&\
lxc exec maas-test -- virsh undefine juju-controller &&\
lxc file push ./controller.xml maas-test/root/ &&\
lxc exec maas-test -- virsh define ./controller.xml &&\
lxc exec maas-test -- virsh start juju-controller
```

Once the VM finishes creation it should be picked up in MAAS under the "Nodes"
tab.

![new node](/img/maas/node-discovery.png)

Click on the node name and then the "Configuration" tab, configure the power
setting for Virsh.

![power-config](/img/maas/power-config2.png)

After saving, use the "Take Action" menu on the top right and choose
"Commission" and then click the green "Commission machine" button to start.

Give MAAS some time and the node should enter the "Ready" state having passed
commissioning. This verifies your PXE process is working and you have a KVM
ready for bootstrapping a juju controller. This was by far the most manual
processes of preparing my home lab. With this setup everything is in place to
deploy software to bare metal and containers with MAAS providing the machines
and juju managing the application configuration.

[maas]: https://maas.io/
[maas-lxd]: https://github.com/chris-sanders/maas-lxd
