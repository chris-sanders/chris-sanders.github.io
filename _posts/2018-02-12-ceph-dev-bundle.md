---
layout: post
title:  "Bundle deploy single node ceph"
subtitle: "Living on the edge"
date:   2018-02-12 20:00:00 +0000
share-img: "/img/ceph_logo.png"
image: "/img/ceph_logo.png"
gh-repo: chris-sanders/bundle-ceph
gh-badge: [star, fork, follow]
tags: [ceph, homelab, charms]
---

I'm going to show you how to deploy a single node Ceph with a juju bundle.  The
charms included in this bundle are all currently a work in progress and should
only be used for testing purposes. I expect changes to be made as I'm getting
them merged into their upstream repositories and I have no plan to provide a
confirmed upgrade path from these edge charms to the upstream branches. They do
all work together and enable a preview of some added features I've been working
on specifically to support single node Ceph in a home lab.

## Deploying

While all of these charms are available on my github I've also pushed them to
the [charm store][charm-store] pre-built with several of the feature branches
merged. If you're interested in the source code all of these charms live on the
*edge* branch of their repositories. If you just want to try out these charms in
their current state, deploying them from the charm store is the best option and
will remain available even if you're reading this some time after it's been
posted.

To make the deployment as easy as possible, I've also created a bundle targeting
the specific revision of these charms used in this post.

|Charm|Charm Store|Github|
|-----|-----------|------|
|Bundle|[cs:~chris.sanders/bundle/ceph][cs-bundle]|[bundle-ceph][gh-bundle] |
|ceph-mon|[cs:~chris.sanders/ceph-mon][cs-ceph-mon]|[ceph-mon][gh-ceph-mon]|
|ceph-osd|[cs:~chris.sanders/ceph-osd][cs-ceph-osd]|[ceph-osd][gh-ceph-osd]|
|ceph-fs |[cs:~chris.sanders/ceph-fs][cs-ceph-fs]|[ceph-fs][gh-ceph-fs]|

To deploy the bundle to the MAAS setup [previously described][maas-blog-post]
run:
```bash
$ juju deploy cs:~chris.sanders/bundle/ceph-0
```

A successful deploy, with 4 Hard Drives on the Ceph node will produce the ```juju
status``` shown below.

```bash
Model    Controller  Cloud/Region  Version  SLA
default  maas-home   maas-home     2.3.2    unsupported

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
If you've made it this far you now have a single node ceph deployed. You can
edit the bundle or charm options and re-deply from scratch very easily. To clean
up you can remove the model:
```bash
$ juju destroy-model default && juju add-model default
```
This will remove all deployed applications so you can redeploy as above with
different configurations. Deploy time is heavily dependent on the boot time for
the hardware as well as your internet connection for downloading and installing
packages.


More details on the charms and the modifications are below.

## The charms

This install will be using the following charms.
 * [ceph-mon][ceph-mon-charm] - Installs a monitor daemon which stores critical
   cluster state required for Ceph daemons to coordinate with each other.
   Monitors are also responsible for managing authentication between daemons and
   clients. 
 * [ceph-osd][ceph-osd-charm] - Installs a Ceph OSD (object storage daemon)
   which stores data, handles data replication, recovery, rebalancing, and
   provides some monitoring information to Ceph Monitors.
 * [ceph-fs][ceph-fs-charm] - Installs a Ceph Metadata Server which stores
   metadata on behalf of the Ceph Filesystem. Ceph Filesystem is a
   POSIX-compliant file system that uses a Ceph Storage Cluster to store its
   data.

For a single node we'll deploy a single ceph-osd directly to a machine which has
four hard drives and is provisioned by MAAS. Setting up MAAS for this task was
covered in a [previous blog post][maas-blog-post] and details about the specific
hardware will be covered at a later time.

Similarly we'll deploy a single Monitor, which we'll put in a LXD container on
the machine running the ceph-osd.  

Finally, we'll deploy three Ceph Metadata Servers, each in a LXD container.  One
for a file system with erasure codes to support a single disk failure, a second
with erasure codes to support two disk failures, and a third which is an active
standby if either of the first two fail.

Next let's review the experimental and in development features which are of
interest for single node home lab use.

## Ceph-OSD

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

### bluestore

The upstream charms have experimental support for Bluestore which was introduced
in [Luminous][new-in-luminous]. While Bluestore is interesting, the key feature
for this use case is that Bluestore OSD's can support erasure encoded
file systems with out requiring a cache layer. Erasure coding provides a
configurable mix of failure domain and storage efficiency.

{: .box-warning}
**12.2.2 Is now available**, this warning is left only to remind anyone that
deployed originally to upgrade.
There is a known race condition in Ceph 12.2.1 with Bluestore. This can corrupt
an OSD. There is a fix in 12.2.2, but as of this writing the Ubuntu Cloud
Archive is using 12.2.1. This is fine for testing and 12.2.2 should be available
soon.

### osd-shared

Shared-osd is a new configuration option I've added which allows for hard drives to be
initialized as an OSD and used by Ceph when they have mounted partitions on
them. This allows you to slice off parts of a drive for things like an mdraid
for the host OS and still use the rest of the drive as a Ceph OSD. This will
affect performance, but I'm willing to trade off IOPS to avoid dedicating two
drives to the host file system.

### osd-reformat

The charms allow you to 'reformat' a drive which was previously an OSD so that
you can cleanly reuse the drive. This option is particularly useful when
deploying and redeploying configurations for benchmarking. There are fixes in
this version of the charm for some edge cases which I found that prevented a
clean re-install with Bluestore OSDs.

### osd-encrypt

The upstream charm allows you to enable encryption but there are issues with
it's use on Bluestore. I've implemented changes which improve this but there is
still a race condition during install as well as a failure to start after
reboot. These can be worked around manually for the purpose of benchmarking but
clearly it's not ready for regular use yet. I'll provide information on how to
recover from these when running benchmarks, leave this option off for now.

## Ceph-mon

The configuration used in this bundle is:
```bash
ceph-mon:
  charm: "cs:~chris.sanders/ceph-mon-0"
  num_units: 1
  expected-osd-count: 12
  bindings:
    "": home-space
    public: home-space
    cluster: home-space
  options:
    source: cloud:xenial-pike
    monitor-count: 1
    osd-failure-domain: True
    allow-multiple-fs: True
  to: 
    - lxd:ceph-osd/0
```

### OSD Failure Domain

I've added this option to set the failure domain to OSD. By default the charm
expects a failure domain of one Ceph node. This means that chunks are not stored
on the same node so the cluster can sustain the loss of a node and still run.
That clearly doesn't apply with single node so this setting makes the failure
domain one hard drive. This is key to making the charm deploy on a single node.

### Allow Multiple FS

I've added support for multiple Ceph Filesystems on the same cluster. The Ceph
documentation states this is experimental, may contain bugs, and definitely
doesn't handle permissions in an expected manor. I'm fine with all of that,
particularly for benchmarking.

## Ceph-FS

The configuration used in this bundle is:
```bash
erasure-single:
  charm: "cs:~chris.sanders/ceph-fs-0"
  num_units: 2
  bindings:
    "": home-space
    public: home-space
  options:
    source: cloud:xenial-pike
    fs-name: 'erasure_1'
    pool-type: 'erasure'
    pool-weight: 60
    config-flags: '{"profile":"single", "erasure-type":"isa", "k":2, "m":1, "failure-domain":"osd"}'
    compression-mode: 'aggressive'
    compression-algorithm: 'zlib'
    compression-required-ratio: .45
  to:
    - lxd:ceph-osd/0
    - lxd:ceph-osd/0
```

### Pool type

The Ceph FS charm has some of the most significant changes from up stream. I've
added support for setting a pool-type for replicated or erasure backed pools.
Along with the pool type the ```config-flags``` setting allows you to set
specific profile settings. Those of most interest include.

 * erasure-type - The erasure algorythm to use, I intend to benchmark the
   standard isa as well as the intel specific implementation.
 * k/m - Sets the erasure k and m values where k is the number of chunks and m
   is the number of parity chunks. This is what allows you to tune the pool for
   the number of drives and amount of parity that suit your needs.
 * failure-domain - Note that we need to set this to OSD for the same reasons as
   above in the Ceph-Mon charm.

I expect the ```config-flags``` setting will change in the future, I'm not
convinced this is the best method for setting these options. It does however let
you set the important settings in one place. After further testing the most
used options will likely end up as standard configuration options instead of
the dictionary string used here.

### Compression

I've added settings that allow you to enable and set the compression mode,
algorithm, and ratio. These are mostly self explanatory and certialy interesting
for benchmarking.

### MDS Hot Spare

You may notice this configuration deploys two of the same Ceph-Fs
configurations. When running Ceph-Fs Ceph expects a spare MDS for failover in
the event that the first becomes unavailable. If there isn't one the Ceph health
will report a warning. Deploying 1 additional Ceph-Fs unit will meet the
requirement for an alternate and any MDS can act as a hot spare regardless of
pool options. Whichever MDS comes online last will sit as an alternate and fill
in for any Ceph-Fs that looses it's primary.

In future posts I'll run tests with various configurations to see how it
performs on my specific hardware. The best way to see how Ceph performs on your
setup is to try it out yourself!

[ceph-osd-charm]: https://github.com/chris-sanders/charm-ceph-osd
[ceph-mon-charm]: https://github.com/chris-sanders/charm-ceph-mon 
[ceph-fs-charm]: https://github.com/chris-sanders/charm-ceph-fs
[maas-blog-post]: https://chris-sanders.github.io/2018-02-02-maas-for-the-home/
[new-in-luminous]: https://ceph.com/community/new-luminous-bluestore/
[charm-store]: https://jujucharms.com/u/chris.sanders
[cs-bundle]: https://jujucharms.com/new/u/chris.sanders/ceph/bundle/0
[gh-bundle]: https://github.com/chris-sanders/bundle-ceph
[cs-ceph-mon]: https://jujucharms.com/new/u/chris.sanders/ceph-mon/0
[gh-ceph-mon]: https://github.com/chris-sanders/charm-ceph-mon
[cs-ceph-osd]: https://jujucharms.com/new/u/chris.sanders/ceph-osd/0
[gh-ceph-osd]: https://github.com/chris-sanders/charm-ceph-osd
[cs-ceph-fs]: https://jujucharms.com/new/u/chris.sanders/ceph-fs/0
[gh-ceph-fs]: https://github.com/chris-sanders/charm-ceph-fs
