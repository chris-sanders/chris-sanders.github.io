---
layout: post
title:  "Bundle Weechat"
subtitle: "IRC made easy"
date:   2019-02-09 20:00:00 +0000
gh-repo: pirate-charmers/bundle-weechat
gh-badge: [star, fork, follow]
share-img: "/img/pirate-charmers/weechat-logo.png"
image: "/img/pirate-charmers/weechat-logo.png"
tags: [charms, pirate-charmers, bundle]
---
Juju bundles define configuration and relations of a group of software. In this
post I'll review a bundle that deploys a secure IRC client and VPN. 
Combining multiple charms in a bundle make this setup very straight forward. 
By the end we'll have a persistent IRC bouncer that can run on the free tier 
of many cloud providers. In the process a VPN will also be setup which can 
be used to access weechat, but can also act as a general purpose VPN to route 
traffic for clients on untrusted networks.

## Deploying
As in previous posts, let's start by getting a deployment started so you can
follow along and have a test environment ready later in the post. I do my test deploys on Google 
Compute, but you can do this deploy on any cloud provider you have setup with juju. 
You can read about [Using GCE with Juju][google-juju] and get started now 
for free. 

Once you have a controller configured you can start the deploy with:

```bash
juju deploy cs:~pirate-charmers/bundle/bundle-weechat
```

The result once settled should look something like this.

```bash
$ juju status --relations
Model      Controller  Cloud/Region     Version  SLA          Timestamp
jaas-test  jaas        google/us-east1  2.4.7    unsupported  16:13:13-06:00

App        Version          Status  Scale  Charm      Store       Rev  OS      Notes
ddclient   0.0.1            active      1  ddclient   jujucharms    1  ubuntu  
haproxy    0.1.4-2-g22f...  active      1  haproxy    jujucharms    2  ubuntu  exposed
weechat    0.0.3-5-g648...  active      1  weechat    jujucharms    3  ubuntu  
wireguard  0.0.2-12-g7a...  active      1  wireguard  jujucharms    1  ubuntu  exposed

Unit          Workload  Agent  Machine  Public address  Ports           Message
ddclient/0*   active    idle   0/lxd/1  252.1.15.7                      
haproxy/0*    active    idle   0        35.190.168.222  80/tcp,443/tcp  
weechat/0*    active    idle   0/lxd/0  252.1.15.100                    
wireguard/0*  active    idle   0        35.190.168.222  15820/udp       

Machine  State    DNS             Inst id              Series  AZ          Message
0        started  35.190.168.222  juju-b3efd1-0        xenial  us-east1-b  RUNNING
0/lxd/0  started  252.1.15.100    juju-b3efd1-0-lxd-0  bionic  us-east1-b  Container started
0/lxd/1  started  252.1.15.7      juju-b3efd1-0-lxd-1  bionic  us-east1-b  Container started

Relation provider     Requirer              Interface     Type     Message
haproxy:reverseproxy  weechat:reverseproxy  reverseproxy  regular  

```
## What was deployed
This bundle deploys four pieces of software.
 * Weechat - An IRC client
 * Haproxy - Provides reverseproxy for Weechat's relay function
 * ddclient - Registers dynamic IP addresses with a domain name to allow
   registering for TLS certificate with Let's Encrypt
 * Wireguard - A fast UDP based VPN

Wireguard and HAProxy are installed on a machine with access to a public IP.
Weechat and ddclient are installed in containers and access is not directly
available externally.

Weechat can be connected to via the Relay or using the VPN to SSH to the weechat
container.

## VPN
### Client setup
To connect to wireguard you'll need wireguard installed on your workstation.
With ubuntu this can be done by installing the PPA and the package:

```bash
sudo add-apt-repository ppa:wireguard/wireguard
sudo apt install wireguard
```

Next we need to configure wireguard to connect to the server we have deployed.
This requires a public and private key, and they can be generated with:

```bash
umask 077
wg genkey | tee pirvatekey | wg pubkey > publickey
```
This will generate two files 'privatekey' and 'publickey' you can `cat` the
files to see your keys.

We will also need some information from the server that we want to connect to,
it can be retrieved with the `get-config` action after Wireguard is deployed.

```bash
$ juju run-action wireguard/0 get-config --wait
  id: eccb1184-54c7-402e-833e-ff7ded453e6a
  results:
    ip: 35.190.168.222
    key: 0gzTG3Iotx+aA4eGjKPdBmHyA2pg11cXfw9PyUvV+Tw=
    port: (15820,)
  status: completed
  timing:
    completed: 2019-02-10 22:20:25 +0000 UTC
    enqueued: 2019-02-10 22:20:24 +0000 UTC
    started: 2019-02-10 22:20:24 +0000 UTC
  unit: wireguard/0
```
We now have all the information to write the config file for our client. Edit
the file `/etc/wireguard/weechat.cfg` replacing the variables with the values
from the above client keys and server config.

```yaml
[Interface]
PrivateKey = $CLIENT-PRIVATE-KEY
Address = 10.10.10.2/32

[Peer]
PublicKey = $SERVER-KEY
Endpoint = $SERVER-IP:$SERVER-PORT
AllowedIPs = 252.0.0.1/8
# AllowedIPs = 0.0.0.0/0
```
The Server values are from the juju command while the client key is from the
files we generated while installing wireguard. I've provided two options for the
AllowedIPs, your choice will depend on how you are using the VPN. This setting
will determine what subnet is routed via the VPN. I've set it to 252.0.0.0 to
match the fan network that the weechat client is on in my cloud (the IP address
that weechat reports in juju status). This allows me to connect to weechat via
the VPN while keeping the rest of my traffic routing normally. If you wanted to
route all traffic to the VPN you could use the other 0.0.0.0/0 to accomplish
that.

### Server setup
The server also has to be configured to allow the client to connect. We need to
tell the server our publickey and the Address that was chosen for the interface.
To do that you can put the config in a yaml file and upload it to the server.
Create a file `peers.yaml` with the following.

```yaml
laptop:
  allowedips: "10.10.10.2/32"
  publickey: "$CLIENT-PUBLIC-KEY"
```
The top level key in the yaml file is a name, and the IP has to match the value
we set in the interface on the client. You can add additional clients as long as
they have unique names and addresses in the same subnet.

Send the configuration to the server with:

```bash
juju config wireguard peers="$(cat ./peers.yaml | base64)"
```
You can verify the peer is listed on the server by running the 'wg' command
on the unit.

```bash
juju run --unit wireguard/0 'wg'
interface: wg0
  public key: 0gzTG3Iotx+aA4eGjKPdBmHyA2pg11cXfw9PyUvV+Tw=
  private key: (hidden)
  listening port: 15820

peer: JCiYyUCYcqdPmw6TIhp6L1vC1exBnWV2Q=
  allowed ips: 10.10.10.2/32
```
The bundle is configured to route traffic from the vpn to device 'ens4'. If you
are deploying on another cloud, you might need to update this for the device
that has your public address. The setting is `forward-dev` and can be set by
passing a string with your device. If your device was eth0 the command would be:

```bash
# This is an example, not needed on GCE
juju config wireguard forward-dev="eth0"
```

### Connect to wireguard

With the server and client configured you can now connect from
the client and see the status with:
```bash
$ sudo wg-quick up weechat
$ wg
interface: weechat
  public key: JCiYyUCYcqdPmw6TIhp6L1vC1exBnWV2Q=
  private key: (hidden)
  listening port: 45234

peer: 0gzTG3Iotx+aA4eGjKPdBmHyA2pg11cXfw9PyUvV+Tw=
  endpoint: $SERVER-IP:15820
  allowed ips: 252.0.0.0/8
```
With the interface up you should be able to ping the weechat container,
which was previously unrechable (252.1.15.100 in the above example). Now that
you can route to the containers which do not have a public address you can
access weechat via VPN/SSH.

## Accessing weechat via VPN/SSH
Weechat has been installed in a container and the weechat service is running in
a screen session of the user `weechat`. You can access the screen session by
sshing to the container, switching to the weechat user, and attaching to the
screen session.

```bash
$ juju ssh weechat/0
ubuntu@juju-b3efd1-0-lxd-0:~$ sudo su - weechat
weechat@juju-b3efd1-0-lxd-0:~$ screen -R
```
The weechat user is a normal user, you can add ssh credentials to this user to
ssh directly to the account if you prefer. As normal you can disconnect from
screen without closing weechat with Ctrl-A-D.

## Accessing weechat via relay
Weechat also supports a 'relay' so that 3rd part applications can connect to it.
An example of a well known client that uses the weechat relay is
[glowing-bear][glowing-bear]. This bundle has setup HAProxy to act as a
reverseproxy for the weechat relay connection. To use it you will have to setup
TLS certificates which can be done with Letsencrypt and ddclient which are
deployed with the bundle.

### Registering Dynamic DNS
For Letsencrypt to issue certificates you need a domain to register with, and
you can set that up with ddclient. Ddclient supports a large number of providers
for updating dynamic addresses. There is a good overview on
[DynamicDNS][ubuntu-dynamic-dns] on help.ubuntu. I'm using [Google
Domains][google-domains], and the ddclient charm is set for that provider by
default. When you create a Dynamic DNS entry in the 'Synthetic records' section
of your domains account you'll be provided a username and password for that
domain.

You can see the available config options on ddclient with:
```bash
$ juju config ddclient
  ddclient-address:
    default: ""
    description: Domain name to register this host to
    source: default
    type: string
    value: ""
  ddclient-login:
    default: ""
    description: Login name
    source: default
    type: string
    value: ""
  ddclient-password:
    default: ""
    description: Password
    source: default
    type: string
    value: ""
  ddclient-protocol:
    default: googledomains
    description: Protocol for the service
    source: default
    type: string
    value: googledomains
  ddclient-server:
    default: domains.google.com
    description: Server to post updates to
    source: default
    type: string
    value: domains.google.com
```
The only things I need to set to use Google Domains is the address, login, and
password. That can be set with:

```bash
$ juju config ddclient ddclient-address="weechat.example.com"
$ juju config ddclient ddclient-login="$LOGIN-NAME-FROM-PROVIDER"
$ juju config ddclient ddclient-password="$PASSWORD-FROM-PROVIDER"
```
The default update interval is 1800 seconds. This means you could have to wait
as long as 30 minutes and DNS can take some time to propagate. Before continuing
you should make sure you can resolve your address with `dig weechat.example.com`
and see that you get back the public IP of your server. Until this is ready you
can not receive a certificate from Letsencrypt.

### Enable Letsencrypt
Once DNS is working, registering with Letsencrypt only requires you tell the
charm what domain to try and register and enable Letsencrypt.

```bash
$ juju config haproxy letsencrypt-domains="weechat.example.com"
$ juju config haproxy enable-letsencrypt=true
```
You can see the result of the registration processes in the debug log:
```bash
$ juju debug-log
```

### Connect via Glowing-bear
With HAProxy now using TLS certificates you can use Glowing-bear to connect to the
relay. You'll access your relay on the domain you registered on port 443 and log
in with the relay password that weechat generated on install. Get the password
with the action `get-relay-password`:

```bash
$ juju run-action weechat/0 get-relay-password --wait
unit-weechat-0:
  id: e5913ba9-0540-4eed-8d2f-493da112d8c9
  results:
    outcome: success
    password: zundeakAV
  status: completed
  timing:
    completed: 2019-02-10 23:23:23 +0000 UTC
    enqueued: 2019-02-10 23:23:22 +0000 UTC
    started: 2019-02-10 23:23:23 +0000 UTC
  unit: weechat/0
```
When connecting with [hosted glowingbear][glowing-bear] be sure to use https not
http and check the 'Encryption' box before connecting. This will ensure your
communication to your weechat instance is encrypted.

## Configure Weechat
You can configure your usernames and other weechat settings which are deatiled
in the [WeeChat quick start guide][weechat-quick-start]. While
you can do this directly in weechat, you can also revision your settings in a
config file and apply them through the charm. This is particularly useful if you
want to spin up temporary weechat clients.

The config format for weechat settings is very simple, you provide one command
per line. For example, you can create the following weechat.cfg file.

```
/server add freenode chat.freenode.net
/set irc.server.freenode.addresses "chat.freenode.net/6667"
/set irc.server.freenode.autoconnect on
/set irc.server.freenode.autojoin "##pirate-charmers"
/connect freenode
```
Apply the file by setting it as the `user-conifg` setting on the weechat charm:
```bash
$ juju config weechat user-config="$(cat ./weechat.cfg)"
```
The file will be applied and settings saved to persist across reboot. If you made it this far
you have been joined to the channel ##pirate-charmers. We would love to hear how you're using 
the charms.

## Customizing the bundle
With a fully operating bundle, you might want save some of the customizations that 
have been applied via the cli. This can be done with an [overlay bundle][overlay-bundle]. 
An overlay bundle applies on top of another bundle to modify it.

As an example, let's use an overlay bundle to include the previously created weechat.cfg
during deploy time. Create the file `weechat-overlay.yaml` with the following:

```yaml
applications:
  weechat:
    options:
      user-config: include-file://weechat.cfg
```

With the overlay defined you could run the deploy with the original bundle and
include the weechat.cfg:

```bash
juju deploy cs:~pirate-charmers/bundle/bundle-weechat --overlay ./weechat-overlay.yaml
```
Using an overlay, you can pre-configure any of the config settings that were set
after deploy. This makes it possible to tear down and re-create your instance
very quickly with all user specific settings kept in an overlay. Just remember,
DNS still takes time to propagate and Letsencrypt shouldn't be enabled during
the deploy. You should check that your domain is available before enabling it.

While this does make it easy to stand up a temporary IRC client and VPN, you
should be aware that Letsencrypt has [rate limits][letsencrypt-limits].
It is very easy to hit the limit of *5 duplicate certificates per week*
redeploying this bundle.

# Closing
With this bundle you have both a persistent IRC bouncer and a UDP based VPN that
can be used to secure mobile clients. The initial deploy performs best with the
default instance size that juju selects but during normal operations
substantially less resources are needed. You can, for example, edit the instance
once it is working as expected to be an f1-mirco on GCE. This size qualifies for
the 'Always Free' tier and as long as your network traffic doesn't exceed normal
limits doesn't cost anything to leave running. This is a great way to stay on
IRC persistently even across multiple clients.

Additionally Wireguard, the vpn deployed, is very efficient for clients that
connect temporarily. This works very well for mobile clients that might
occasionally be on an untrusted WiFi network. The UDP nature of Wireguard means
that clients don't need to send a keep alive, only transmitting when a
connection is needed. This saves both on data as well as battery since the device
can sleep while on the VPN. Wireguard also handles roaming clients well.
Wireguard servers will use the unique key for each client to update the last
known IP address of the client, allowing clients to roam between
connections to the server without issue.

If you're interested in Wireguard and not IRC you can just deploy Wireguard and
use the VPN section of this blog to configure peers.

[taskd-github]: https://github.com/pirate-charmers/layer-taskd
[JAAS]: https://jujucharms.com/jaas
[glowing-bear]: https://glowing-bear.org
[ubuntu-dynamic-dns]: https://help.ubuntu.com/community/DynamicDNS
[google-domains]: https://domains.google.com
[google-juju]: https://docs.jujucharms.com/2.4/en/help-google
[weechat-quick-start]: https://weechat.org/files/doc/stable/weechat_quickstart.en.html
[overlay-bundle]: https://docs.jujucharms.com/2.5/en/charms-bundles#overlay-bundles
[letsencrypt-limits]: https://letsencrypt.org/docs/rate-limits/
