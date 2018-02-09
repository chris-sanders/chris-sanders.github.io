---
layout: post
title:  "Bundle deploy single node ceph"
subtitle: "Living on the edge"
date:   2018-02-07 20:00:00 +0000
share-img: "/img/ceph_icon.webp"
image: "/img/ceph_icon.webp"
gh-repo: chris-sanders/bundle-ceph
gh-badge: [star, fork, follow]
tags: [ceph, homelab, charms]
---

Today I'm going to show you how to deploy a single node ceph with a juju bundle.
The charms included in this bundle are all currently a work in progress and
should only be used for testing purposes. I expect changes to be made as I'm
getting them merged into their upstream repositories and I have no plan to
provide a confirmed upgrade path from these edge charms to the upstream
branches. They do all work together and enable a preview of some added features
I've been working on specifically to support single node ceph in a home lab.

## The charms

This install will be using the following charms.
 * [ceph-mon][ceph-mon-charm] - Installs a monitor daemon which stores critical
   cluster state required for Ceph daemons to coordinate with each other.
   Monitors are also responsible for managing authentication between daemons and
   clients. 
 * [ceph-osd][ceph-osd-charm] - Installs a ceph osd (object storage daemon)
   which stores data, handles data replication, recovery, rebalancing, and
   provides some monitoring information to Ceph Monitors.
 * [ceph-fs][ceph-fs-charm] - Installs a Ceph Metadata Server which stores
   metadata on behalf of the Ceph Filesystem. Ceph Filesystem is a
   POSIX-compliant filesystem that uses a Ceph Storage Cluster to store its
   data.

For a single node we'll deploy a single ceph-osd directly to a machine which has
four hard drives and is provisioned by MAAS. Setting up MAAS for this task was
covered in a [previous blog post][maas-blog-post] and details about the specific
hardware will be covered at a later time.

Similarly we'll deploy a single Monitor, which we'll put in a LXD container on
the machine running the ceph-osd. While operating with a single monitor is
possible, it's very unsafe. 

{: .box-warning}
:warning: If you loose all monitors you **loose all data** in your ceph cluster.
Technically the data is still there, but you loose all of the metadata that told
you *where* it was. It's effectively gone.

Finally, we'll deploy three Ceph Metadata Servers, each in a LXD container.  One
for a file system with erasure codes to support a single disk failure, a second
with erasure codes to support two disk failures, and a third which is an active
standby if either of the first two fail.

Next let's review the experimental and in development features which are of
interest for single node home lab use.

## New Features

### Ceph-OSD

The configuration used in this bundle is:
```yaml
ceph-osd:
  charm: "cs:~chris.sanders/ceph-osd-0"
  num_units: 1
  bindings:
    "": home-space
    public: home-space
    cluster: home-space
  options:
    source: cloud:xenial-pike
    autotune: True
    bluestore: True
    osd-shared: True
    osd-devices: "/dev/sda /dev/sdb /dev/sdc /dev/sdd"
    osd-reformat: "Yes"
    osd-encrypt: False
```

#### bluestore

The upstream charms have experimental support for Bluestore which was introduced
in [Luminous][new-in-luminous]. While Bluestore is interesting, the key feature
for this use case is that Bluestore OSD's can support erasure encoded
file systems with out requiring a cache layer. Erasure coding provides a
configurable mix of failure domain and storage efficiency.

{: .box-warning}
There is a known race condition in Ceph 12.2.1 with Bluestore. This can trigger
and corrupt an OSD. There is a fix in 12.2.2, but as of this writing the Ubuntu
Cloud Archive is using 12.2.1. This is fine for testing and 12.2.2 should be
available soon.

#### osd-shared

Shared-osd is a new configuration option I've added which allows for hard drives to be
initialized as an OSD and used by ceph when they have mounted partitions on
them. This allows you to slice off parts of a drive for things like an mdraid
for the host OS and still use the rest of the drive as a Ceph OSD. This will
affect performance, but I'm willing to trade off IOPS to avoid dedicating two
drives to the host file system.

#### osd-reformat

The charms allow you to 'reformat' a drive which was previously an OSD so that
you can cleanly reuse the drive. This option is particularly useful when
deploying and redeploying configurations for benchmarking. There are fixes in
this version of the charm for some edge cases which I found that prevented a
clean re-install with Bluestore OSDs.

#### osd-encrypt

The upstream charm allows you to enable encryption but had there are issues with
it's use on Bluestore. I've implemented changes which improve this but there is
still a race condition during install as well as a failure to start after
reboot. These can be worked around manually for the purpose of benchmarking but
are not yet stable. I'll provide information on how to recover from these when
running benchmarks, leave this option off for now.


```bash
Model    Controller  Cloud/Region  Version  SLA
default  maas-home   maas-home     2.3.1    unsupported

App             Version  Status   Scale  Charm     Store       Rev  OS      Notes
ceph-mon                 waiting    0/1  ceph-mon  jujucharms    0  ubuntu
ceph-osd                 waiting    0/1  ceph-osd  jujucharms    0  ubuntu
erasure-double           waiting    0/1  ceph-fs   jujucharms    0  ubuntu
erasure-single           waiting    0/2  ceph-fs   jujucharms    0  ubuntu
ntp                      waiting      0  ntp       jujucharms   24  ubuntu

Unit              Workload  Agent       Machine  Public address  Ports  Message
ceph-mon/0        waiting   allocating  0/lxd/3                         waiting for machine
ceph-osd/0        waiting   allocating  0        192.168.1.212          waiting for machine
erasure-double/0  waiting   allocating  0/lxd/0                         waiting for machine
erasure-single/0  waiting   allocating  0/lxd/1                         waiting for machine
erasure-single/1  waiting   allocating  0/lxd/2                         waiting for machine

Machine  State    DNS            Inst id  Series  AZ       Message
0        pending  192.168.1.212  6hbfym   xenial  default  Deployed
0/lxd/0  pending                 pending  xenial
0/lxd/1  pending                 pending  xenial
0/lxd/2  pending                 pending  xenial
0/lxd/3  pending                 pending  xenial

Relation provider   Requirer                 Interface  Type         Message
ceph-mon:mds        erasure-double:ceph-mds  ceph-mds   regular      joining
ceph-mon:mds        erasure-single:ceph-mds  ceph-mds   regular      joining
ceph-mon:mon        ceph-mon:mon             ceph       peer         joining
ceph-mon:osd        ceph-osd:mon             ceph-osd   regular      joining
ceph-osd:juju-info  ntp:juju-info            juju-info  subordinate  joining
ntp:ntp-peers       ntp:ntp-peers            ntp        peer         joining
```

```bash
Model    Controller  Cloud/Region  Version  SLA
default  maas-home   maas-home     2.3.1    unsupported

App             Version       Status  Scale  Charm     Store       Rev  OS      Notes
ceph-mon        12.2.1        active      1  ceph-mon  jujucharms    0  ubuntu  
ceph-osd        12.2.1        active      1  ceph-osd  jujucharms    0  ubuntu  
erasure-double  12.2.1        active      1  ceph-fs   jujucharms    0  ubuntu  
erasure-single  12.2.1        active      2  ceph-fs   jujucharms    0  ubuntu  
ntp             4.2.8p4+dfsg  active      1  ntp       jujucharms   24  ubuntu  

Unit               Workload  Agent  Machine  Public address  Ports    Message
ceph-mon/0*        active    idle   0/lxd/3  192.168.0.246            Unit is ready and clustered
ceph-osd/0*        active    idle   0        192.168.1.212            Unit is ready (4 OSD)
  ntp/0*           active    idle            192.168.1.212   123/udp  Ready
erasure-double/0*  active    idle   0/lxd/0  192.168.0.105            Unit is ready (1 MDS)
erasure-single/0   active    idle   0/lxd/1  192.168.0.245            Unit is ready (1 MDS)
erasure-single/1*  active    idle   0/lxd/2  192.168.0.104            Unit is ready (1 MDS)

Machine  State    DNS            Inst id              Series  AZ       Message
0        started  192.168.1.212  6hbfym               xenial  default  Deployed
0/lxd/0  started  192.168.0.105  juju-46f714-0-lxd-0  xenial  default  Container started
0/lxd/1  started  192.168.0.245  juju-46f714-0-lxd-1  xenial  default  Container started
0/lxd/2  started  192.168.0.104  juju-46f714-0-lxd-2  xenial  default  Container started
0/lxd/3  started  192.168.0.246  juju-46f714-0-lxd-3  xenial  default  Container started

Relation provider   Requirer                 Interface  Type         Message
ceph-mon:mds        erasure-double:ceph-mds  ceph-mds   regular      
ceph-mon:mds        erasure-single:ceph-mds  ceph-mds   regular      
ceph-mon:mon        ceph-mon:mon             ceph       peer         
ceph-mon:osd        ceph-osd:mon             ceph-osd   regular      
ceph-osd:juju-info  ntp:juju-info            juju-info  subordinate  
ntp:ntp-peers       ntp:ntp-peers            ntp        peer      
```

[ceph-osd-charm]: https://github.com/chris-sanders/charm-ceph-osd
[ceph-mon-charm]: https://github.com/chris-sanders/charm-ceph-mon 
[ceph-fs-charm]: https://github.com/chris-sanders/charm-ceph-fs
[maas-blog-post]: https://chris-sanders.github.io/2018-02-02-maas-for-the-home/
[new-in-luminous]: https://ceph.com/community/new-luminous-bluestore/
