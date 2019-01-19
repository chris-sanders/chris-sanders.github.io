---
layout: post
title:  "Pirate-charmers"
subtitle: "setting sail"
date:   2018-09-19 20:00:00 +0000
share-img: "/img/pirate-charmers/pirate-charmers.png"
image: "/img/pirate-charmers/pirate-charmers.png"
tags: [homelab, juju, charms]
---

Several years ago I set down a path to move my personal infrastructure into juju
control. Along the way I've written several charms and blog posts about the
things I've learned and developed. Today I'm happy to announce I'm
ending my solo effort and will be joined by James aka [ec0][ec0-blog].

# Pirate-Charmers
To transition from a single owner to a group my current charms will will be moving to the 
namespace pirate-charmers both [on github][pirate-charmers-github] as well as 
[on the charm store][pirate-charmers-charmstore]. New charms will be started
directly in this name space.

Github has a very nice redirect system, so that change will be 
transparent to  most readers. The charmstore however, to my knowledge, has no way of addressing
such a move. If you currently use any of the charms I maintain today, I
recommend moving to the same charm maintained by pirate-charmers when it becomes
available. Switching your deployed charms source location can be done with 
[the switch flag][jujucharms-switch] as documented on jujucharms.com.

## Pirates quest
The goal remains the same, James and I both find charms to be a great method
for deploying and operating software. We're still focused on a smaller home lab type environment. 
While charms for big data and data center scale
tooling are becoming more common we're looking to address the smaller projects
that we use ourselves. We'll certainly leverage components that are already
written, but when necessary will write our own charms when the functionally we
need is not well addressed by current offerings.

All of the charms we write for pirate-charmers will be designed to work
together. The aim will generally be to enable the specific functionally
we need today and expand it in the future as we add additional software. A good example 
of this is the [HAProxy charm][haproxy-charm]. The first use case for this charm
was to handle reverse-proxy and TLS certificates for an environment. With new
charms that are under development right now it is being expanded to better
address tcp back ends as well as letting a single charm register more than one
reverse-proxy configuration.

Over time each charm should become more feature complete, but the initial goal
will be to support the small or home lab environment for specific tasks. Remember  
we both use these charms ourselves to run our own services so we are greatly
incentivized to start with our desired feature set and expand rather than wait
for the perfect charm.

## On the horizon
### Charms
As mentioned above, the HAProxy charm will be seeing a new release on
pirate-charmes to make it better support any type of other workload. The current
version has already been migrated to pirate-charmers and this version will be
updated when the new functionally is merged into the master branch.

In addition to HAProxy, charms for ddclient and Weechat have also been pushed to the
charm store. With these three charms today, you can stand up weechat on your own
domain name, register TLS certificates, and use an encrypted relay for
web/mobile clients. This has been in development for a few weeks, and while I am
not yet running it full time I have used it as a backup install and expect to
move to it full time soon.

Finally, James has taken on a massive first effort to join in the fun by
charming Gitlab. Gitlab will also be deployable behind HAProxy, including
forwarding ssh ports. Enabling communication and code hosting seemed like a good
place to start this journey.

We don't currently have, and likely won't try to maintain a specific roadmap.
Generally speaking, we're going to implement the things we want to run. There
might be a reasonable way for us to map out a rough idea and make that known but
it's not a high priority right now.

## Bundles
As the number of charmed services increase we would like to provide an easy way
to kick-the-tires and demonstrate the charms. That will likely come in the form
of pre-defined bundles that allow you to quickly stand up related services. The
above mentioned weechat setup is an example, and Gitlab itself requires several
components to get running. By providing a bundle we can easily show how it works
and you can integrate that into your environment.



[ec0-blog]: ec0.io
[pirate-charmers-github]: https://github.com/pirate-charmers
[pirate-charmers-charmstore]: https://jujucharms.com/u/pirate-charmers/
[jujucharms-switch]: https://docs.jujucharms.com/2.4/en/developer-upgrade-charm
[haproxy-charm]: https://jujucharms.com/u/pirate-charmers/haproxy
