---
layout: post
title:  "Installing ceph-osd in a container"
subtitle: "Thinking inside the box"
date:   2018-05-11 20:00:00 +0000
share-img: "/img/lxd_logo.png"
image: "/img/lxd_logo.png"
tags: [ceph, lxd, homelab]
---

Containers are a great tool for delivering and operating software. There are
some situations that are difficult to run in containers including applications
which require access to block devices. LXD provides a way to mount block
devices into a container allowing applications like ceph-osd to run inside a
LXD container. This post will describe how I'm using LXD to containerize
ceph-osd, the benefits it provides, and some caveats you should be aware if you
go down a similar path.

## Why LXD 
I started exploring this configuration because I'm running a single
node Ceph server in my home lab. Details are available in other blog posts, if
you're interested in the details of the setup you can [read more][read more]
on those posts.

I've made changes to several of the charms used to deploy Ceph to make them more
home lab friendly. Those changes have not yet received reviews ([feedback
welcome][gerrit]). I want a path forward with my current changes and a
reasonable expectation to upgrade or switch back to the upstream charm at a
later date. For most of the setup installing in LXD containers allows a rolling
upgrade process. Both ceph-mon and ceph-fs deploy in containers with no special
configuration. The exception has been ceph-osd which needs to access the block
devices.

By installing ceph-osd in a LXD container I've been able to migrate drives
between different versions of ceph-osd charm. While this isn't a guarantee of
forward compatibility, as long as the ceph-osd charm can recognize and use
existing ceph block devices the rest of the charm can be completely changed and
with no compatibility requirements for the charm. Additionally, ceph-osd can be
reinstalled without releasing and redeploying the bare metal which would wipe
some of the drives during reinstall.

## The setup

### LXD Profile

LXD Provides [documentation][device configuration] on configuring devices for
LXD containers. For this configuration, you'll need to pass through the block
devices, loop devices, and loop-control. Additionally, several allowances need
to be made for modifying the block devices. This will remove much of the
isolation that a container normally provides. In this case the container is not
being used for security reasons so that's fine but be aware if using a similar
setup for other use cases.

The profile I'm using for this is:

```yaml
config:
  raw.lxc: |-
    lxc.aa_profile=unconfined
    lxc.cgroup.devices.allow = b 8:* rwm
    lxc.cgroup.devices.allow = b 7:* rwm
    lxc.cgroup.devices.allow = c 10:237 rwm
    lxc.mount.auto=
    lxc.mount.auto=sys:rw proc:mixed cgroup:mixed
  security.privileged: "true"
devices:
  loop-control:
    path: /dev/loop-control
    type: unix-char
  loop0:
    path: /dev/loop0
    type: unix-block
  loop1:
    path: /dev/loop1
    type: unix-block
  loop2:
    path: /dev/loop2
    type: unix-block
  loop3:
    path: /dev/loop3
    type: unix-block
  loop4:
    path: /dev/loop4
    type: unix-block
  loop5:
    path: /dev/loop5
    type: unix-block
  loop6:
    path: /dev/loop6
  loop7:
    path: /dev/loop7
    type: unix-block
  md0:
    path: /dev/md0
    type: unix-block
  sda:
    path: /dev/sda
    type: unix-block
  sdb:
    path: /dev/sdb
    type: unix-block
  sdc:
    path: /dev/sdc
    type: unix-block
  sdd:
    path: /dev/sdd
    type: unix-block
```

Notice that not just the block devices that Ceph will be using are being passed
through. I've included an mdraid device as well. This is because the ceph-osd
charm will error when device are listed as mounted but not available to the
system. Be sure all devices are made available to the container.

### UDEV rules

Passing the devices through to the container allows them to be used by ceph-osd
but there will still be a problem when modifying the partitions on the devices.
The host machine will create the corresponding filesystem node but they will
not be mapped into the container. In order to allow ceph-osd to create and
remove partitions a set of udev rules is used to respond to the *add* and
*remove* actions creating the nodes in the container.

Place the following udev script in the container at: /etc/udev/rules.d/99-lxd-mnt.rules

```bash
# Skip items we don't want to deal with
ACTION=="changed", GOTO="persistent_storage_end"
SUBSYSTEM!="block", GOTO="persistent_storage_end"
KERNEL!="sd*", GOTO="persistent_storage_end"
ENV{DEVTYPE}!="partition", GOTO="persistent_storage_end"

# Add partitions
KERNEL=="sd*", ACTION=="add", RUN+="/bin/sh -c '/bin/echo /bin/mknod $env{DEVNAME} b $env{MAJOR} $env{MINOR} >> /root/udev.log 2>&1'"
KERNEL=="sd*", ACTION=="add", RUN+="/bin/sh -c '/bin/mknod $env{DEVNAME} b $env{MAJOR} $env{MINOR} >> /root/udev.log 2>&1'"

# Remove partitions
KERNEL=="sd*", ACTION=="remove", RUN+="/bin/sh -c '/bin/echo rm $env{DEVNAME} >> /root/udev.log 2>&1'"
KERNEL=="sd*", ACTION=="remove", RUN+="/bin/sh -c 'rm $env{DEVNAME} >> /root/udev.log 2>&1'"

LABEL="persistent_storage_end"
```

These rules are logging any add and remove events in a log file in
/root/udev.log for debugging purposes. You can remove that after you have
confirmed creating and removing partitions work as expected. The number of add
and remove events should not be large so it's fine to leave in place as well.
With the udev rules in the container the ceph-osd charm can fully manage the devices
that have been passed through.

## Notes on use with ceph-osd

I have been running my fork of ceph-osd inside a container to test this
configuration for more than a month. I have added, removed, and replaced disks
while migrating data onto the node and dealing with failed drives. During that
time everything operated as expected.

### Adding devices

When adding devices the device must be added to the host machine and once the
/dev/sdX device is available on the host the ceph LXD profile can be modified
to add the new device. The container will have the device added live, no reboot
is required. This allows you to stage devices on the host, manually clear them,
and do burn in testing before adding them to the ceph LXD.

### Removing devices

Removing devices requires additional thought. If you are permanently removing
the device you must also remove it from the LXD profile. LXD will not start a
container if a device is defined and not available to mount in the container.
The container will continue to run however if a device which was mounted during
boot is no longer available. Additionally, since devices are mounted by their
/dev/sdX path you must account for the device renaming when the host is
rebooted. The order of the devices does not matter since ceph will identify the
drives by the osd id on the drive itself.

To reduce the number of devices permanently, shut down the ceph LXD, remove
the last device from the profile, reboot the host, and verify the ceph-osd
charm has the reduced number of devices and started normally.

The easiest way to replace a failed device is to follow the steps above to
remove a device and then the steps to add a new device. If you would like to
try replacing the failed device without rebooting, remove the failed drive,
add the replacement, and verify the device received the same /dev/sdX as the
removed device. You may have to reapply the profile to the lxd container for
the drive to be mounted into the container.

### Reinstalling / Migrating charm

The ceph-osd charm will import drives that are setup for Ceph and match the
existing pool this makes migrating the drives to a new container straight
forward. Passing drives through to a new ceph-osd container will import the
drives with no user intervention needed. After adding new devices to the
container a restart of the container will re-scan the drives. The charm will
recognize the devices as OSD devices and start the osd processes, Ceph will show
the osds move to the new host as they check in with the ceph-mon. After all of
the osds are recognized on the new host the old host, which should have no osds,
can be purged. The only thing common to the two charms are the block devices,
there is no chance for compatibility issues between charm versions.  Any
ceph-osd charm that recognizes the devices will work.

A helpful hint while testing this setup, you can poke the ceph-osd charm to make
it rescan devices without rebooting. This could change in future charms so I
only recommend you use this if you're familiar with the charm or using it during
testing. Calling the hook ```storage.real``` will cause a rescan.
```bash
juju run "./hooks/storage.real" --service=ceph-osd
```

### Restrictions

I have not tested this with journal devices or bcache. Both configurations add
additional complexity to identifying devices and may not work with the provided
udev and profile. Also, if using journal devices be aware moving devices between
ceph nodes requires the journal and the block device be moved together this will
apply to moving devices between two ceph-osd containers as well.

I have tested encryption and this configuration does not support cryptsetup
inside the container. This is not a limitation of the ceph-osd charm but a
conflict with the kernel driver inside a LXD container. If a configuration
which allows cryptsetup to work from within a container is found the ceph-osd
charm should work. As above this adds additional complexity to the drive/device
mappings and may require additional care when adding, removing, or replacing
drives. Do not try to containerize encrypted drives with my fork of the
ceph-charms and the above configurationg.

If you know how to get cryptsetup working in a LXD I am interested to hear about
it, I simply haven't had time to dig into the issue.


[read more]: https://chris-sanders.github.io/tags/#ceph
[gerrit]: https://review.openstack.org/#/q/owner:sanders.chris%2540gmail.com+status:open
[device configuration]: https://github.com/lxc/lxd/blob/master/doc/containers.md#devices-configuration

