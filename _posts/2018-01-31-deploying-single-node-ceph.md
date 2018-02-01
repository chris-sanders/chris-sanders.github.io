---
layout: post
title:  "Deploying Single node ceph"
subtitle: "A managed approach"
date:   2018-01-31 20:00:00 +0000
image: "/img/ceph_icon.webp"
tags: [ceph, homelab, maas, juju]
---

If you're going to be running Ceph you'll quickly find that you do not want to
deploy it by hand. There are several daemons involved and even on a single node
the processes isn't as easy as just installing a few packages. In some ways
installing on a single node is less intuitive since you'll have processes and
daemons split out specifically to allow them to scale and be redundant and then
running them all right next to each other.

## The tools
To address the installation process and make it highly repeatable I'll be using
several of the cloud technologies from Canonical. They're all open source, which
is going to be important since this use case is **not** mainstream by any
stretch of the imagination and changes will be needed. Additionally, I like them
so that counts for something.

### MAAS
The first thing I'll need is a server with hard drives. I have no interest in
watching an OS install so [MAAS][maas] is definitely going to be in the mix.
Also I know that MAAS works well with some of the other tooling. It does have
some requirements to do it's thing, but I wanted a machine with IPMI for remote
management anyway. With used enterprise hardware available today at reasonable
prices there is no reason to buy something without remote management built in.
If you don't care about Meltdown and Specter I bet there's about to be a
lot more hardware on the market soon! 

### Application Management
Once I have a server I need a way to manage the software and configuration. I've
looked at Ansible and Saltstack before and they're a good fit for what they do,
but for what I'm doing I prefer to use [juju][juju]. This is a highly
opinionated subject and if your preference is something else that's won't offend
me one bit. I'll briefly explain why *I* prefer using juju.

#### Ansible / Salt / Configuration Management
I find some of the modern configuration management systems are very quick
to get up and running and very easy to make changes. They typically have built
in support for most common tasks from installing software to editing a config
file. However, once you get into complex relations I've found they quickly
devolve into custom configuration scripts which are not self contained and
usually not easy to share with others. You still end up managing the
configuration of every application explicitly. While you can do this centrally
most solutions rely on configuration from one software matching a setting in
another and it quickly becomes a house of cards.

Additionally, I haven't seen great solutions for dealing with staged install
processes. For example, when I salted Nginx with Letsencrypt I didn't find a
good solution to setting that up in one shot. The solutions I found and
contributed to always required you to do an install, register, and then make
another configuration change to use the now available certificates. It was
always multi-stage and so you needed an external 'playbook' or notes to remind
yourself the order of operations to make it work right.

#### Application Modeling with Juju
Juju solves both my main complaints by having a system of state based events and
a method for software to define an interface to pass information over. This
means each charm (the script that manages software) is self contained. It knows
about itself and the relations it makes available or uses, but doesn't need to
worry about configuration of any other software. This keeps configurations for
one software out of another which I find much easier to manage, understand, and
develop.

The state based nature of juju also allows for single shot install of even
complex application configurations. Using the Nginx example above, providing an
option to register with Letsencrypt is self contained. Due to the state
dependent nature, if I ask for the Letsencrypt option on my Nginx charm it will
install Nginx without a certificate, register with Letsencrypt, add the new
certifications to the config, and reload all on it's own nothing else to
remember. One caveat for anyone getting ahead of me, I actually don't think
there is an Nginx charm today with Letsencrypt. I have written a 
[HAProxy charm][haproxy-charm] with Letsencrypt which I very much enjoy and have
plans to continue improving after this Ceph project.

There is a downside to using juju and charms of course. It's not as fast to
learn and understand. The added complexity also means it can take longer to
write the charms and test them to work in the variety of scenarios it should. I
find these trade-offs worth it. When I'm done my plan is to have the changes I'm
making for this project accepted upstream. I intend to upload forked versions to
the charmstore in the interim so you can try it out even while I'm working to
get the changes accepted. In fact, that's a post I intend to make soon&trade; so
you can try it out on your hardware.

### Following along
If you're interested in following along you're in luck I have several more post
which will get into the technical details now that I've outlined the plan. If
you're interested in seeing how Juju works *right now* or want to start learning
so you can better follow the more technical posts I recommend running through
the [Juju and LXD][juju-and-lxd] Getting started page. All you need is the
Ubuntu machine you're reading this post on and you can deploy locally.

[maas]: https://maas.io/
[juju]: https://jujucharms.com/
[haproxy-charm]: https://jujucharms.com/u/chris.sanders/haproxy
[juju-and-lxd]: https://jujucharms.com/docs/stable/tut-lxd
