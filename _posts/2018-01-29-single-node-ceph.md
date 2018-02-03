---
layout: post
title:  "Single node ceph"
subtitle: "Probably a terrible idea"
date:   2018-01-29 20:00:00 +0000
share-img: "/img/ceph_icon.webp"
image: "/img/ceph_icon.webp"
tags: [ceph, homelab]
---

Make no mistake about it, running ceph on a single node is a strange decision.
This is a brief overview of why I'm interested in doing it.

I've been running [Unraid][unraid] since some time in 2008 as a home media
server/NAS. At that time, Unraid was a fairly minimal storage solution which had
one very specific use case that it excelled at; cost effective storage optimized for write
once read many with reasonable failure protection.

I won't go into a full review of Unraid, but it is important to note that it's
performed very well for me over the years. The biggest complaint I have is the
lack of file level check sums. With Unraid if the parity is found to be
incorrect, there isn't enough information to determine if the error is on the
parity drive or data drive. This on *one* occasion was the direct cause of 
data loss for me. Since that time I've added a ZFS mirrored pool for my most important
data. However, I still use Unraid for media storage for things I can replace
even if it's a bit of a pain.

That brings me to this project. I've long wanted to find a storage solution that
would give me the easy add-any-drive expansion and cost effectiveness of
Unraid with better data protection like ZFS.

### Comparing Solutions

Let's see how Ceph, in this use case, compares to ZFS and Unraid for a home
storage solution. Note ZFS has a lot of options each with different benefits, I'm
specifically limiting the comparison to mirrored or Raid Z2. I'll use an initial 6
disk setup to compare.

| Feature | Unraid | Ceph  | ZFS Mirror | ZFS Raid Z2 |
| ------  | :----: | :---: | :--------: | :---------: |
| Minimum Expansion | 1 disk | 1 disk | 2 disk | 6 disk |
| Maximum Expansion | 1 machine | No practical limit  | 1 machine | 1 machine |
| Parity Cost | 2/6 (33%) | 2/6 (33%) | 3/6 (50%) | 2/6 (33%) |
| Fault tolerance | 1 disk | 2 disk | 2 - 3 disks | 2 disk |
| Drive Sizes | Any | Any | 2 x Matched | 6 x Matched |
| Complexity | Low | Unbelievable | Medium | High |

A couple of caveats for the Unraid solution. I'm quoting this with a cache drive
in use, this means your two largest drives have to be the parity and cache.
While this isn't mandatory I've found write performance without a cache to be
very low and a warm spare is great so for my use case it's the only way I run
Unraid anymore.

With the above, for my use case, Ceph stacks up very favorably to the other
options. With the use of [erasure coding][erasure-coding] Ceph can be configured
with very cost effective redundancy.

### The challenge

I've not spoken about performance at all in the above comparison.
That's partially because for my use case on a home server it's almost a
footnote. It is however still interesting and there are levels of performance
that would be unsuitable. From the above solutions, Unraid is expected to be
bottom runner for performance and having used it for years I'm fairly confident
the other solutions will be sufficient for me. As I proceed I will do some
performance testing to verify before I fully commit. Ceph single-node
benchmarks are all but non-existent online today and estimating performance of
these solutions on various hardware is a non-trivial task even with solid data
to start from.

From the comparison above, there is one major downside to Ceph over the other
solutions I've used previously. The level of complexity is not trivial.
Fortunately for me, I have another interest which happens to coincide with this
project. I've been converting my home infrastructure to be fully version
controlled and modeled in code. My preferred method of doing that today is using
[juju][juju] with [MAAS][maas] and [LXD][lxd]. Since Ceph is a major part of
OpenStack there is already a basis for running Ceph with these tools that is
maintained by Canonical. The target audience today is not single node home
installs. 

This blog is going to document my attempts to fix that. Probably a terrible
idea, but I need to see that for myself.

[unraid]: https://lime-technology.com
[erasure-coding]: http://docs.ceph.com/docs/master/rados/operations/erasure-code/
[juju]: https://jujucharms.com/docs/stable/getting-started
[maas]: https://docs.ubuntu.com/maas/2.3/en/
[lxd]: https://linuxcontainers.org/lxd/

