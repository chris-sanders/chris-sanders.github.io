---
layout: post
title:  "K8s metal ingress with TLS"
subtitle: ""
date:   2020-09-10 23:30:00 +0000
gh-repo: chris-sanders/ansible_k14_deployment
gh-badge: [star, fork, follow]
share-img: "/img/kubernetes/kubernetes.png"
image: "/img/kubernetes/kubernetes.png"
tags: [k8s, tools, homelab]
---
[Carvel][carvel] (formerly k14s) combined with Ansible to orchestrate a git-ops workflow has
after a lot of testing become my preferred k8s deployment technique. This post will cover how
to use the k14 Ansible role to deploy the foundation of an edge/homelab k8s cluster.

# The goal
This post is going to cover two topics. First, I'm going to be demonstrating how to use 
[ansible_k14][ansible_k14]. This is a git-ops focused work flow that I'm using to completely
define a deployment which can be checked into git for review and then be deployed into a
cluster. The way I'm approaching this, I commit both the code that generates the deployment
resources as well as the full set of resources that it generates. A configuration control
purist may take issue with checking in the *output*, and that's fair it isn't for everyone.
I'll demonstrate the benefits checking in these files provides and why I think it outweighs
the disk space cost of a handful of text files.

The second goal of this post is to demonstrate how to setup what I consider table stakes for
edge/homelab k8s infrastructure. Even if the deployment technique is not your cup of tea, the
example should still provide useful as an example how to configure several pieces of software that
are required to make a k8s cluster useful.

# Services
Many Kubernetes guides cover the use of cloud hosted clusters for deploying applications.
However, I want to cover edge/homelab configurations. This means using k8s to deploy on a
small cluster without relying on cloud provider applications. To make a cluster useful, it
needs to address secure ingress and routing.

## Metallb
The first building block that is necessary is a LoadBalancer. Kubernetes does not provide a
native solution for LoadBalancers, relying on cloud providers to implement it externally. For
a edge/homelab setup Metallb provides a Network LoadBalancer that allows services to work on
your bare metal deployment like it would a cloud provider. If you aren't familiar with
Kubernetes and LoadBalancers, this provides a static IP address for your services for external
traffic routing.

## Traeifk
[Traefik][traefik] provides a reverse proxy and load balancing, in Kubernetes terms it's an
Ingress controller. While metallb provides the static IP address, Traefik handles routing
including routing based on HTTP headers, direct TCP passthrough, and UDP load balancing.
Comparing Ingress options, very few support UDP routing and even less allow for some advanced
processing like adding HTTP Basic Authentication.

## Cert-manager
If you're going to run services with external access, you need valid TLS certificates. There's
no excuse to run services without TLS thanks to Letsencrypt and [cert-manager][cert-manager]
provides certificates for your k8s services supporting several certificate providers. This
example will set up letsencrypt, but cert-manager can be used to issue other certificates if
you want to as well.

## Acme-dns
[Acme-dns][acme-dns] was not initially part of my design, but it quickly became a necessity.
Letsencrypt now supports wildcard certificates, but it requires DNS authentication. Acme-dns
provides a service that runs in the k8s cluster to enable DNS authentication allowing you to
issue any certificates, including wildcard certificates, for your services.

## Putting it together
With the above, the local firewall will have rules for ports 80, 443, and UDP 53 routed to
static addresses managed by Metallb. Traefik will manage the routing of the requests to
services, and will use a valid TLS certificate. The TLS certificate will be registered and
kept up to date by cert-manager, supported by acme-dns allowing registration of subdomains,
including wildcard certificates.

# Performing the deployment
## Setup k8s
If you don't have a k8s cluster, my preferred cluster for homelab/edge is
[microk8s][microk8s]. You can run it locally as a single node, and as of the 1.19 release you
can easily cluster multiple nodes for a light weight, pure upstream, HA kubernetes cluster.

To get started install microk8s and enable a few addons.
```bash
sudo snap install microk8s --channel=1.19/stable --classic
sudo microk8s.enable dns rbac storage
sudo microk8s.config > ~/.kube/config && sudo chown $USER:$USER ~/.kube/config
```
This enables a few add-ons including storage. If you want a multi node cluster [add the extra
nodes][clustering] with a couple of additional commands.

Verify the cluster is installed and you have the `~/.kube/config` working with:
```bash
$ kubectl get namespaces
NAME              STATUS   AGE
kube-system       Active   10m
kube-public       Active   10m
kube-node-lease   Active   10m
default           Active   10m
```

Clone the example repository:
```bash
git clone https://github.com/chris-sanders/ansible_k14_deployment.git
cd ansible_k14_deployment/sites/site2
```
To use the deployment scripts included in the repository, we need to create a namespaces for
kapp to store deployment information in. Do that with:
```bash
$ kubectl create namespace kapp
namespace/kapp created
```
## Software requirements
The example deployments below can be performed using the existing resources in the manifest
folders for the applications. The examples are using [kapp][kapp] for deployment, although you
can use `kubectl` directly if you prefer. This means most of the following examples can be run
without any requirements, although completing the full configuration to use a letsencrypt
certificate will require some configuration customization and generating configuration
specific to your site requires the following software.

 * [ansible][ansible] - required to run the roles providing the work flow automation
 * [ytt][ytt] - for template processing 
 * [kbld][kbld] - for image dereferencing during template processing
 * [kompose][kompose] - to generate templates from docker-compose files
 * [helm3][helm] - to generate templates from helm charts
 * git - for source control and retrieving some of the above templates

On Ubuntu you can install Carvel, Kompose, Helm, and Ansible with:
```
# Carvel for updating configation and deployment
wget -O- https://k14s.io/install.sh | bash

# Kompose for updating config
curl -L https://github.com/kubernetes/kompose/releases/download/v1.21.0/kompose-linux-amd64 -o kompose
chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose

# Helm3 for updating config
sudo snap install helm --classic

# Ansible for updating config
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```
## Deploying
Each service can be deployed via the `deploy.sh` script in it's folder. To get the initial
deployment up and running, we can deploy everything by calling all of the deploy scripts.
From within the folder `sites/site2` loop over and deploy with:
```bash
for d in */; do (cd $d; ./deploy.sh); done
```
Since the scripts deploy with `kapp` you can get a list of deployed applications with:
```bash
$ kapp -n kapp list
Target cluster 'https://192.168.1.108:16443' (nodes: x1c)

Apps in namespace 'kapp'

Name          Namespaces              Lcs   Lca
acme-dns      (cluster),acme-dns      true  2m
bitwarden     (cluster),bitwarden     true  2m
cert-manager  (cluster),cert-manager  true  1m
metallb       (cluster),metallb       true  1m
traefik       (cluster),traefik       true  1m

Lcs: Last Change Successful
Lca: Last Change Age

5 apps

Succeeded
```
This shows that kapp has deployed 5 apps. The `Lcs` indicates if the last change was
successful (all were), and `Lca` tells you the age of the last change where all of my changes
were within the last few minutes.

Verify that all of the pods are started across all applications:
```bash
$ kubectl get po --all-namespaces
NAMESPACE      NAME                                       READY   STATUS      RESTARTS   AGE
kube-system    hostpath-provisioner-5c65fbdb4f-nrdvx      1/1     Running     3          36h
kube-system    calico-kube-controllers-847c8c99d-f5jkx    1/1     Running     3          36h
kube-system    coredns-86f78bb79c-t6f62                   1/1     Running     3          36h
kube-system    calico-node-w5fll                          1/1     Running     9          36h
acme-dns       acmedns-69f7d9557-8q8w7                    1/1     Running     0          91s
bitwarden      bitwarden-6769cf7776-wzht6                 1/1     Running     0          87s
bitwarden      bitwarden-http-test                        0/1     Completed   0          72s
cert-manager   cert-manager-cainjector-57dd8c88c9-xccrr   1/1     Running     0          58s
cert-manager   cert-manager-6f7456b9d6-5gc8k              1/1     Running     0          57s
cert-manager   cert-manager-webhook-6d64dd847-gdmxr       1/1     Running     0          57s
metallb        metallb-controller-f6466b866-nc6s9         1/1     Running     0          45s
metallb        metallb-speaker-862r2                      1/1     Running     0          45s
traefik        traefik-556fd5f7c6-2vlp9                   1/1     Running     0          22s
```

## Traefik
This deployment uses two external IP's one for UDP and one for HTTP routing.  The address
range was configured on Metallb. Additionally, this deployment included a route to expose the
Traefik Dashboard, protected with Basic Authentication.

To view the dashboard, you need to route the address "dashboard.traefik" to `service/traefik`
. Look up the external IP's that were assigned to traefik services.
```bash
$ kubectl get service -n traefik
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
traefik-udp   LoadBalancer   10.152.183.133   10.0.9.1      53:30174/UDP                 8m50s
traefik       LoadBalancer   10.152.183.139   10.0.9.2      80:32476/TCP,443:30795/TCP   8m50s
```
This shows the standard service is on `10.0.9.2` and the upd services is on `10.0.9.1`. The
external IP range was set in the `site.yaml` for site2 under the metallb configuration.
Customizing these values can be done through the site file and will be described later.

The easiest way to view the dashboard with a single node test cluster is to add
"dashboard.traefik" to your hosts file. Edit `/etc/hosts` to add dashboard and dns, which
will be used later for acme-dns:
```bash
10.0.9.2  dashboard.traefik
10.0.9.2  dns.traefik
```
If you're wondering accessing these addresses is being handled by iptables rules managed by
Kubernetes. You won't see the IP address on any of your interfaces.
Now open a browser to `https://dashboard.traefik` and use the username/password
'traefik' when promoted. You will have to accept the insecure connection because the
TLS is not yet setup. You should see a dashboard similar to this:

![traefik-screenshot](/img/kubernetes/traefik-screenshot.png){:.img-shadow .img-rounded}

## Acme-dns
Switch to the acme-dns folder and deploy with the deployment script.
```bash
cd acme-dns
./deploy.sh
```
You can verify that the pods are up and running in the acme-dns namespace:
```bash
$ kubectl get all -n acme-dns
NAME                           READY   STATUS    RESTARTS   AGE
pod/acmedns-75b8866bc8-48wn7   1/1     Running   0          14s

NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
service/acmedns   ClusterIP   10.152.183.134   <none>        443/TCP,53/TCP,53/UDP,80/TCP   14s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/acmedns   1/1     1            1           14s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/acmedns-75b8866bc8   1         1         1       14s
```
Traefik also installed a route for `dns.traefik` which will route to the acme-dns service. If
you added the dns.traefik name to your `/etc/hosts` earlier you should be able to test the
acme-dns service by registering via curl.
```bash
$ curl -X POST http://dns.traefik/register
{"username":"32e49783-4365-42bb-a3fc-9fa99b583a70","password":"7P9AHF_XA_ki336QmA7LnrXBfwTHAKUujYFk5Xai","fulldomain":"4ab84fab-d51b-48d5-bda8-dcb68f7eb029.dns.traefik","subdomain":"4ab84fab-d51b-48d5-bda8-dcb68f7eb029","allowfrom":[]}
```
The result of the above curl shows a successful registration and returns the credentials
necessary to update the service for DNS records, which is what cert-manager will need to
complete the acme-dns challenge. Copy the reply and keep it for later.

## Cert-manager
Switch to the cert-manager folder and deploy with the deployment script.
```bash
cd cert-manager
./deploy.sh
```
You can verify that the pods are up and running in the cert-manager namespace:
```bash
$ kubectl get all -n cert-manager
NAME                                          READY   STATUS    RESTARTS   AGE
pod/cert-manager-ddd798568-q24hk              1/1     Running   0          35s
pod/cert-manager-cainjector-6f5ddb997-r2kw5   1/1     Running   0          35s
pod/cert-manager-webhook-9984d57c-xvckn       1/1     Running   0          34s

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.152.183.80    <none>        9402/TCP   35s
service/cert-manager-webhook   ClusterIP   10.152.183.154   <none>        443/TCP    35s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           35s
deployment.apps/cert-manager-cainjector   1/1     1            1           35s
deployment.apps/cert-manager-webhook      1/1     1            1           35s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-ddd798568              1         1         1       35s
replicaset.apps/cert-manager-cainjector-6f5ddb997   1         1         1       35s
replicaset.apps/cert-manager-webhook-9984d57c       1         1         1       35s
```
At this point all of the necessary services are setup, and we can now configure Traefik to
request a Certificate from cert-manager. 

# Enable Letsencrypt DNS verification
Using the registration information from acme-dns the configuration of cert-manager can be
updated to perform DNS challenges. This will require some external configuration for your
domain provider. Additionally, Traefik's configuration will be changed to request a
certificate for your domain instead of using a self signed certificate.

The configuration updates will require the software listed above. At this point the
configuration must be customized to match your environment. Following a git-ops work flow the
site settings will be updated, new manifests will be generated, and after you review the
changes the new manifests applied.
# Automation 

## Work flow
The work flow is designed to generate configurations to be checked into revision control. This
makes reviewing changes easy as both the input and resulting output can be seen in a diff.
Additionally everything needed to deploy or update an application is in the repository. Each
commit is a fully deployable configuration.
The steps that will be executed are:
 * Retrieve the templates for a service
 * Generate kubernetes objects from the templates
 * Generate any additional supporting objects from the application role
 * Apply overlay modifications to resources if they are defined
 * Resolve image references to their digest form (immutable)
 * Write files to site/application specific folder for review and deployment

The process allows for multiple sites with site specific configuration as well as application
specific overlays if customization is needed on a per-site basis. The example repository has
two sites in the sites folder. Site 2 will be used in this example, it doesn't include
encryption while Site 1 shows what the configuration looks like when using [sops][sops] for
encryption of secrets.

## Directory Structure
The final directory structure for a site and application will follow the following scheme:
```bash
sites/
└── site1
    ├── application
    │   ├── deploy.sh
    │   ├── diff.sh
    │   ├── manifest
    │   │   ├── ConfigMap.yaml
    │   │   ├── daemonset.yaml
    │   │   ├── deployment.yaml
    │   │   ├── namespace.yaml
    │   │   ├── rbac.yaml
    │   │   └── service-accounts.yaml
    │   ├── secrets
    │   │   └── secrets.yaml
    │   └── overlays
    │       └── manifest.yaml
    └── site.yaml
```
The role generates:
 * application folder
 * deploy and diff scripts
 * manifest folder populated with K8s objects
 * secrets folder with a secrets file (if used by the application role)

The site.yaml file is used for site specific settings and is not generated by the role.
The overlays folder and contents are not generated and are used to apply site specific
customizations to an application if they exist.


## Configure domain provider

## Update configuration

## Generate new manifests

## Review and apply

## For Later
```yaml
dashboard:
    hostname: dashboard.traefik
    users: dHJhZWZpazokYXByMSR0NjRUdDJDdCRxSDRsWTlna0tnL2ZJVkJrUDlPdjku
```
The hostname key configures the address that traefik will route to the dashboard service and
the users key configures the username/password for basic authentication. The base64 value to
set for the user key can be generated with:
```bash
$ echo -n $(htpasswd -nb traefik traefik) | base64
```
## Ceph-csi-rbd
[Ceph][ceph] is the open source storage solution for data centers. A k8s cluster without
storage isn't a terribly useful cluster. You can use host storage for following along with
this post, but I'll include an example of configuring ceph for storage. 


[ansible_k14]: https://github.com/chris-sanders/ansible_k14
[deployment]: https://github.com/chris-sanders/ansible_k14_deployment
[carvel]: https://k14s.io/
[metallb]: https://metallb.universe.tf/
[traefik]: https://containo.us/traefik/
[acme-dns]: https://github.com/joohoi/acme-dns
[cert-manager]: https://github.com/jetstack/cert-manager
[ceph]: https://ceph.io/
[kapp]: https://get-kapp.io/ 
[ytt]: https://get-ytt.io/
[kbld]: https://get-kbld.io/
[kompose]: https://github.com/kubernetes/kompose
[helm]: https://helm.sh/docs/intro/install/
[ansible]: https://www.ansible.com/
[sops]: https://github.com/mozilla/sops
[microk8s]: https://microk8s.io/docs
[clustering]: https://microk8s.io/docs/clustering
