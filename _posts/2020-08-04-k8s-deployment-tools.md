---
layout: post
title:  "Deploying K8s applications"
subtitle: "How I'll be skinning the cat"
date:   2020-08-04 23:30:00 +0000
gh-repo: chris-sanders/ansible_k14_deployment
gh-badge: [star, fork, follow]
share-img: "/img/kubernetes/kubernetes.png"
image: "/img/kubernetes/kubernetes.png"
tags: [k8s, tools, homelab]
---
Kubernetes is well established and the variety of tools to work with it can be overwhelming.
This post will provide an overview of how I'm planning to deploy workloads on K8s for my
home lab. I'll review the primary tool set that I've found the most useful, explain what I
find so compelling about it, and show some examples of how I'm using it today. At the end I'll
provide some thoughts of the other tools I reviewed if you're interested to see why I'm not
using them today.

# K14s
The [k14s][k14s] suite of tools isn't a tool but a group of tools that can be used together.
The project has a simple description "Kubernetes Tools that follow the Unix philosophy to be
simple and composable."

The simplicity and flexibility of this set of tools is ultimately what made it stand out for
me. Each tool is written to solve a specific problem and I find they solve that problem in a
very clean fashion. The developers can be found in the [#k14s][slack] channel of the Kubernets
slack. In my experience someone is regularly available to help you get started.  I've seen on
more than one occasion where features or bug fixes have been identified based on actual users
use cases from discussions on slack. A developer community that is taking feedback and actual
use cases from real world users is a very good sign.

The main disadvantage I found with k14s was the lack of an example workflow using the tools.
After some experimentation I'm currently using Ansible and [the repository linked to this
post][k14_deployment] provides a working example. I'll write a follow up post specifically
about how this repository works with Ansible it's to much to cover in this post.

## ytt
[Ytt][ytt] was the tool that first introduced me to the k14s tools. It's ability to handle templates
as well as overlays make it a useful tool in almost any Kubernetes work flow.

### Templates
The stand out feature of ytt is that it works directly with YAML structures, not text. This
means you can write templates and overlay files without having to deal with indention levels
and unintuitive quoting. At first glance that may sound like a small change, but it makes
templates much cleaner and easier to read and write. It also includes a sandboxed Pythonic
language which makes template logic very easy to read. Here is an example of a template I'm
using to generate ingress objects.

```yaml
#@ load("@ytt:data", "data")
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-tls-dashboard
  namespace: #@ data.values.traefik.namespace
spec:
  entryPoints:
    - websecure
  routes:
    - match: #@ "Host(`{}`)".format(data.values.traefik.dashboard.hostname)
      kind: Rule
      #@ if data.values.traefik.dashboard.users:
      middlewares:
      - name: auth
      #@ end
      services:
      - name: api@internal
        kind: TraefikService
  tls: {}
#@ if data.values.traefik.dashboard.users:
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: auth
  namespace: #@ data.values.traefik.namespace
spec:
  basicAuth:
    secret: authsecret
#@ end
#@ if hasattr(data.values.traefik, "ingress_routes"):
#@ for app in data.values.traefik.ingress_routes:
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: #@ "traefik-tls-{}".format(app.name)
  namespace: #@ app.namespace
spec:
  entryPoints:
    - websecure
  routes:
    - match: #@ "Host(`{}`)".format(app.uri)
      kind: Rule
      services:
      - name: #@ app.service_name
        port: #@ app.service_port
        namespace: #@ app.namespace
  tls: {}
#@ end
#@ end
```
If you're familiar with Kuberenets you can probably with no previous ytt experience
understand this template. The syntax for ytt is based on annotating YAML files with comments
that inject data. I find this a very natural way to customize YAML. There are a few advanced
examples in this template as well. You'll notice that if I configure the data value
`data.values.traefik.dashboard.users` a Middleware will be defined and used.  Additionally, at
the end an IngressRoute is created for each app that I define in the
`data.values.traefik.ingress_routes` list. You can see the final result in the
[ingress][ingress-yaml] file in the traefik deployment. Data values come from the [site
file][site-file].

Both the readability and easy of implementing new templates in ytt are significant
improvements over any other templating tool I've reviewed. I know it doesn't seem like the
world needs another way to write templates, but I think ytt gets it right for Kubernetes.

### Overlays
The ability for ytt to merge a YAML file with an overlay is also very powerful and something I
found lacking in most deployment tools. Starting from a Helm Chart is a great way to quickly
deploy software on K8s. However, Charts can only support so many configuration options and you
will eventually find a small alteration that you need which is not supported by and may never
be supported by the upstream chart. Overlays solve this problem by allowing you to maintain
adjustments in an overlay without having to fork and maintain a new Chart.

Overlays are not as straight forward to read/understand as a template is but they are
very powerful. Here is an example overlay I'm currently using from the
ceph-csi-rbd deployment.
```yaml
#@ load("@ytt:overlay", "overlay")

#@overlay/match by=overlay.subset({"kind": "DaemonSet"}), expects="0+"
---
spec:
    template:
        spec:
            containers:
            #@overlay/match by="name"
            - name: driver-registrar
              args:
              #@overlay/match by=overlay.index(2)
              - --kubelet-registration-path=/var/snap/microk8s/common/var/lib/kubelet/plugins/rbd.csi.ceph.com/csi.sock
            #@overlay/match by="name"
            - name: csi-rbdplugin
              volumeMounts:
              #@overlay/match by="name"
              - name: plugin-dir
                mountPath: /var/snap/microk8s/common/var/lib/kubelet/plugins
              #@overlay/match by="name"
              - name: mountpoint-dir
                mountPath: /var/snap/microk8s/common/var/lib/kubelet/pods
            volumes:
            #@overlay/match by="name"
            - name: socket-dir
              hostPath:
                  path: /var/snap/microk8s/common/var/lib/kubelet/plugins/rbd.csi.ceph.com
            #@overlay/match by="name"
            - name: registration-dir
              hostPath:
                  path: /var/snap/microk8s/common/var/lib/kubelet/plugins_registry
            #@overlay/match by="name"
            - name: plugin-dir
              hostPath:
                  path: /var/snap/microk8s/common/var/lib/kubelet/plugins
            #@overlay/match by="name"
            - name: mountpoint-dir
              hostPath:
                  path: /var/snap/microk8s/common/var/lib/kubelet/pods
```
In this example, I'm applying an overlay to a DaemonSet that ceph-csi deploys as part of it's
Helm Chart. The goal here is to replace some paths so that the deployment works with microk8s
which locates the /var/lib folder inside the snap path. The `match by="name"` statements are
identifying which item in a list I'm modifying and then the keys are merged with the original
document. In this case I'm changing `mountPath` and `path`. This modification is enough to
allow me to use the chart even though it doesn't support this use case yet. You can see the
final result with the replaced variables in the [nodeplugin-daemonset][node-daemonset].

## kbld
Kbld is a tool to help manage images for your deployments. I'm specifically using it for it's
ability to resolve image references to their digest form. What that means is that I can use a
Helm Chart that uses an image tag and kbld will change the tag to the digest form explicitly
listing the image that matches that tag at the time I create my files. The first example on
the [kbld][kbld] website shows what this looks like. 

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
  annotations:
+    kbld.k14s.io/images: |
+      - Metas:
+        - Tag: 1.17
+          Type: resolved
+          URL: nginx:1.17
+        URL: index.docker.io/library/nginx@sha256:2539d4344...
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
-        image: nginx:1.17
+        image: index.docker.io/library/nginx@sha256:2539d4344...
        ports:
        - containerPort: 80
```
You can see an example on the [nodeplugin-daemonset][node-daemonset], which uses a tag in the
Helm Chart but is listed in the digest form after processing.

This is very helpful to track images that might receive security updates without changing the
tag. Using tags your pod would not be updated when the image is updated because it still
matches the tag you originally deployed with. Using the digest, the image reference in the
deployment will change when the Chart is processed if a new image was released for the tag.

## kapp
[Kapp][kapp] is a deployment tool, it focuses on installing your resources into the cluster.
Kapp adds a few things that kubectl alone doesn't address. The items I found most compelling
were:
 * Converges application resources (creates, updates and/or deletes resources) in each deploy
based on comparison between provided files and live objects in the cluster
 * Separates calculation of changes (diff stage) from application of changes (apply stage)
 * Creates CRDs and Namespaces first and supports custom change ordering

The first  bullet point means that every time I run the deploy only modified items are updated
and if I removed a CRD it is removed. Every deploy brings the cluster in alignment with the
configuration on disk preventing drift. The second bullet pairs well with this by supporting a
diff command to see what has changed as well as allowing you to review changes before
committing them to the cluster. The third item is a matter of convenience by supporting
resource ordering I can keep all resources together and apply the whole folder without
failures due to resource ordering. 

From my personal experience testing a variety of tools I've found that kapp tracks each
application's resources better than most deployment tools. Some deployment tools don't do a
good job of removing *everything* that was installed if you decide to delete a deployed
application. Kapp does a good job of removing everything, including cluster resources and
namespaces which I've frequently seen leftover after an install.

You can see kapp being used in the `deploy.sh` scripts in the repository.
```bash
# Decrypt secrets and deploy
sops -d secrets/secrets.yaml | \
kapp deploy -a bitwarden \
-n kapp \
--into-ns bitwarden \
-f manifest \
-y \
-f -
```
Every time this script is run the deployment is reconciled updating things that change and
removing resources that were removed. To delete this whole application you can run `kapp
delete -n kapp -a bitwarden` and everything including cluster wide resources and the namespace
will be removed.

## sops
[Sops][sops] is not part of k14s but due to the design of the k14s tools it can be used with
the k14s tools. This is an example of the benefit that k14s gains by using small focused tools
that allow you to easily integration the portions you need.

Sops is created my Mozilla and describes itself as "an editor of encrypted files that
supports YAML, JSON, ENV, INI and BINARY formats and encrypts with AWS KMS, GCP KMS, Azure Key
Vault and PGP."

Sops, like ytt, operates on YAML structures allowing it to encrypt all or only specific values
in a YAML file. This lets me use sops to encrypt secrets on disk decrypting them during
deployment. Keeping the secrets encrypted and revisioned with the rest of the resources allows
the repository to fully describe the deployment no additional information needs to be tracked.
PGP support means I can add private keys for specific developers or CI pipelines allowing
me to select who (or what) can access the secret portions in a very straight forward fashion.

Below is an example of a ceph-csi-cephfs secrets configuration generated with a template that
uses sops.
```yaml
apiVersion: v1
kind: Secret
metadata:
    name: csi-cephfs-secret
    namespace: ceph-csi-cephfs
stringData:
    userID: ENC[AES256_GCM,data:8KcvQKk=,iv:kB9zgrHpubZ0ovqK1lmgJGIEGnjnKc6PS7hk3gqDq14=,tag:lEFFzgSKy/I/6svDMmbH1A==,type:str]
    userKey: ENC[AES256_GCM,data:sPex/g1ohtNKr37neOlL38HkFCZnVhKBpoTcF+P59SJPPORrqJN76g==,iv:GWX12DaD6/WW+qhYBLxXeoZz/CXnFZdzj5DO9JGmdqI=,tag:XD0uJnaoDizs9d2FpnTLOQ==,type:str]
    adminID: ENC[AES256_GCM,data:x8uGW6c=,iv:2MNp3tWNsz6BkibBi41wnMQB5QeVYmgFg5pjMIJOV6A=,tag:w+mOjoVSitqpa8OfMrTV6A==,type:str]
    adminKey: ENC[AES256_GCM,data:0z7nrBPblfx0eqRSOYArpnXv9aJFTc1xa63tB50k6l7GK/NFPlw1rg==,iv:4l3f+l/NOiiWyYmlQFO3y3akXpp+4WYpx5wvjRLpaIM=,tag:C0ZJWHJIM9UGGaRaubYaqg==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    lastmodified: '2020-08-02T19:52:57Z'
    mac: ENC[AES256_GCM,data:CRY5DrccLiObBCRW/UT3jEjKpJeq2NGsmeN31TQU61zIprkhuFJ+lS3AWxJjySvfWlJ6XWdHLoFXLsgU8TSvSmySzZkqoxSZrHRRWAb7LYIM+Tp0OrG5pn1glsmg9J2TvzyDqrObl3ug7D0JcTIN70RKk8NMxbBr7d+MM8BcNRs=,iv:5G7NbWx3AC1zDjIE/uoLC137+jWH2yaw/TI+Y+MyeV4=,tag:BhhvtMmyKxtW4UuTLqXh5A==,type:str]
    pgp:
    -   created_at: '2020-08-02T19:52:57Z'
        enc: |-
            -----BEGIN PGP MESSAGE-----
            wcFMA8+hman75/nUARAABsY3AxLaqWnyvT0yJpSBWsyOYtzzQYG4loFekA7rBVoz
            VNgFj34VdyYTAkZBZGfrHBASueKFUHi2whwKhRTi95BhMnl5BTPOqgObRBuaHCMg
            K1t2kT6ACOKqqhj+QnWW5F6vaf8whDidYv4lA/W9msM4XydAyKIPgWvuCyDvlxel
            cX+p3mcI51x0XLXkcHBTSi0yhU8F4sEodpCTaUJXmQwSZA4AGxb7/R6p8jUHmgrB
            CyJQGclS7NekXK7G/cmUcDaVD6P3ljYXg+yXGDCer16tc6uWcOlmRKGlVHFNJp73
            xPojRbHFDw83owZZGuPEZAo4nXChq/EHQB8+2LMO91mWYz6c5530muPK9A9bvuRB
            3N34EMDwnRuqfoQWB/Vkfr9g7Kl58Iaf+Kfc4OEGi6swYnEBRgOCcoE2mn3ZVFsH
            f22MlOUxu7Kz1BahJTlPQlnb25JBSDmlTJMKDLYdFRyWNyTBb8OMxYGEc76NUk0f
            9ZUb9BvrukPgWQ6I+GPrdYU1+ureo7YnZknqD3bD7O+xH5eGo7fI6ofAniZ1lyvN
            cdWIFv3Xmn4faeKzhxSKUC4tcgq5z5xRpiLf4CIFmd1J205QQXc1tgn0qRIk2FH/
            HrRF/W9GUc+njvsJ0ukzDU9PYnoZYkdavKKXoY+i4tfyr7oIQXEXpo5zjczeU3jS
            4AHkMoSphMG7unBSN6WFPaqu4eGFPOAk4PjhWDPgtuLCingP4DLlRbIq3Q4BfWRH
            fjL7yTDPVP2hVl7Csba5Hqudq9pxpV7gR+T8UhmGtPuOgPSByphPAjN34pbH/K/h
            v0IA
            =QhV9
            -----END PGP MESSAGE-----
        fp: 211FA1AE58F975F266A3E42E3CF239734CDBFD58
    encrypted_regex: (.*secret.*|.*password.*|.*user.*|.*admin.*)
    version: 3.5.0
```
Specific values are being encrypted before written to disk, sops also supports encrypting all
values in a YAML file. I'm not using that here, but it's probably a better default to avoid
accidentally leaking secrets. There is a `sops` section of the file where the sops
configuration was written, this allows sops to operate on the file without needing any other
configuration.

## kapp-controller
[Kapp-controller][kapp-controller] is another k14s project, but not one that I'm currently
using. It's an in-cluster service to deploy applications as part of a CD process. There is
also an interesting [issue][kapp-local] tracking the intention to run kapp-controller via CLI.
I'll be watching this tool it could replace the Ansible I'm using today.

# The other tools
Here are some of the other items I reviewed and run sample deployments with before selecting
k14s. For your use case you might find some of these tools a better fit.

## Kapitan
[Kapitan][kapitan] was one of my favorite tools and one I might still use in the future if
some of the feature requests that I've made are implemented. Kapitan is intended to be a
generic template management and generation system. It address having a way to organize input
data and generate output in an idempotent fashion. I've written some simple deployments with
Kapitan and it is very DRY allowing you to combine components together to manage kubernetes
deployments across sites and applications.

Why am I not using it? Currently Kapitan doesn't support multiple build steps to address things
like overlays. I've found starting from a Helm Chart often requires an overlay to unblock me
*right now* and Kapitan doesn't have a solution to that yet. I have filed a bug [to request
overlay support][kapitan-overlay] and I may replace my current Ansible with Kapitan when I can
get the same results with it. My only other complaint is I just can't warm up to jsonnet. I
simply don't like it. There is an active issue on the project to [include ytt][kapitan-ytt] as
a template engine. If the overlay issue is solved I would likely look at implementing the ytt
feature. Humans shouldn't have to write json it's terribly unfriendly to *write*, it's fine
for data exchange.

I hope Kapitan can eventually replace the Ansible I'm using today. It would be faster, easier
to implement complex logic, and easier to troubleshoot.

## Juju
I've been using juju for a number of years now for my bare metal and LXD based deployments.
Brining the application modeling paradigm to K8s applications is something I'm very interested
in. After trying with Juju I've found a few issues that make it unsuitable for my use today.
Currently, juju doesn't allow charms to model [ingress objects or cluster
resources][juju-ingress]. Additionally, while testing CAAS charms I triggered several issues
with Juju and K8s becoming out of sync.  This gets better with each release, but with the
2.7.x (2.7.5 I think) release that I did my testing for this comparison with it wasn't stable
enough at that time for me to select it.

I'll revisit Juju and application modeling again in the future. If you're interested in trying
it out, check out the [juju discourse][juju-discourse] where you'll find the latest
information on working with Juju and Kubernetes. The official [juju documentation][juju-docs]
is good to get started if you are new to Juju, but with the Kubernetes support quickly
changing you'll find many features you want for CAAS charms documented on discourse.

## Helm
[Helm][helm] shouldn't need an introduction. As shown above I'm building my templates on top
of Helm Charts where available to expedite deployments. Helm is nice, but even with Helm 3 I
prefer having the ability to provide perform post processing and I like storing my configuration on
disk. Helm just seems best fit as part of the solution not the only tool.

## Ansible
Ansible has a [k8s module][ansible-k8s] which lets you configure and deploy with Ansible.
While this does benefit from using the Ansible inventory system for variables if you use
Ansible to directly deploy your files there is no good way to reconcile changes like Helm and
Kapp. Similar to writing Ansible playbooks for anything else the author needs to write the
install and uninstall steps. The potential for untracked items to be left in the cluster over
time makes this an unattractive option for anything but ephemeral test clouds. You can fully
describe your configuration in Ansible but rewriting Kubernetes objects into Ansible YAML
decouples you from using existing Charts.

I am using Ansible as a work flow automation today, but it doesn't seem like a very viable
deployment method.

## Rancher
I deployed [Rancher][rancher] but I didn't attempt to use the CLI. Rancher provides a web
based way to deploy and manage Kubernetes clusters as well as workloads on the clusters.
Installing applications in a cluster consists of selecting from Helm Charts either built in
or from custom repositories you can configure.

While Rancher might be very good at what it does, it's kind of the opposite of the k14s
approach. Rancher addresses cluster installation, user / project management, application
deployments, mutli-site management, and even CI Pipelines. To install it on my bare metal test
cluster I first had to configure a load balancer (metallb), ingress (traefik), and storage
(ceph). This meant I needed a way to configure and deploy these initial services outside of
Rancher and once I have that with k14s I didn't see value in using Rancher for the other
deployments.

If you find Rancher compelling check out [Lense][lense] as well it has a similar feel to it.

# Closing
I'll post a follow up specifically about how I'm using the [k14 role][ansible_k14] to quickly
generate and update deployments in the [example repository][k14_deployment]. This post is less
hands on than what I normally post, but I felt in this case it was worth writing based on the
large number of tools I've reviewed and tested. Hopefully you've found an interesting tool to
add to your kit of Kubernetes tricks.

The landscape of deployment tools is vast. I didn't even cover everything I tried out much
less everything I've read about. If you've got a favorite tool that makes working with
Kubernetes better let me know I'm sure I've missed some gems out there.

[k14s]: https://k14s.io/
[slack]: https://slack.kubernetes.io/
[kapitan]: https://kapitan.dev/
[k14_deployment]: https://github.com/chris-sanders/ansible_k14_deployment
[ansible_k14]: https://github.com/chris-sanders/ansible_k14
[kbld]: https://get-kbld.io/
[ytt]: https://get-ytt.io/
[kapp]: https://get-kapp.io/
[sops]: https://github.com/mozilla/sops
[kapitan-overlay]: https://github.com/deepmind/kapitan/issues/527
[kapitan-ytt]: https://github.com/deepmind/kapitan/issues/354
[juju-docs]: https://juju.is/docs
[juju-discourse]: https://discourse.juju.is/
[juju-ingress]: https://discourse.juju.is/t/caas-charms-installing-resources-and-secrets/1997/15
[helm]: https://helm.sh
[ansible-k8s]: https://docs.ansible.com/ansible/latest/modules/k8s_module.html
[rancher]: https://rancher.com/docs/rancher/v2.x/en/overview/architecture/
[kapp-controller]: https://github.com/k14s/kapp-controller
[kapp-local]: https://github.com/k14s/kapp-controller/issues/2
[node-daemonset]: https://github.com/chris-sanders/ansible_k14_deployment/blob/master/sites/site1/ceph-csi-cephfs/manifest/nodeplugin-daemonset.yaml
[ingress-yaml]: https://github.com/chris-sanders/ansible_k14_deployment/blob/master/sites/site1/traefik/manifest/ingress.yaml
[site-file]: https://github.com/chris-sanders/ansible_k14_deployment/blob/master/sites/site1/site.yaml
[lense]: https://github.com/lensapp/lens
