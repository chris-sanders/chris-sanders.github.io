---
layout: post
title:  "Charm Create"
subtitle: "Where to start"
date:   2019-01-28 20:00:00 +0000
gh-repo: pirate-charmers/charm-template
gh-badge: [star, fork, follow]
share-img: "/img/board-cubes.png"
image: "/img/board-cubes.png"
tags: [charms, pirate-charmers, charm-template]
---
Charms are very flexible and can be written several ways and even in different
languages. Having an opinionated template limits those options but gets you
started faster. This post will cover a template we'll be using for the
pirate-charmers charms that get you started with unit and functional testing out
of the box. By the end of this post you'll be able to quickly create a template
that passes testing so you can focus on charming and not test frameworks.

I'll perform the install inside a LXD container from scratch. If you want to do
the same to kick the tires you'll need a container that supports nesting. Create
the container with:

```bash
lxc launch ubuntu -c security.nesting=true -c security.privileged=true
lxc exec $YOUR_CONTAINER_NAME -- sudo su - ubuntu
```
Once the container has started you can use the second command to become ubuntu
in the container. You can do this without a privileged container [here is an example][container-blog]
 with more information. 

# Charm Tools
Templates are available via the charm-create command which comes with the charm
tools snap necessary for building charms. Today that tools doesn't support
external 3rd party templates. The feature is being reviewed to allow this
support but until that time I'm working around the issue with a monkey patch on
the tool to enable a custom template.

If you don't already have the charm tools you should install the snap.

```bash
sudo snap install charm --classic
```
Verify the install by running charm create help

```bash
charm create -h
```
This should output the help from charm-create.

Next you'll need the [charm-template][charm-template] repository to provide the
monkey patch. This is not the repository [for the template][template-repo], this is
the monkey patch which should go away if/when charm-tools are able to natively
include external templates. You do not need to edit or download the template
repository the tool will do that when you create a new charm.

```bash
git clone https://github.com/pirate-charmers/charm-template.git
cd charm-template
```
Now verify the monkey patch is working by running the charm-create.py from the
repository and verifying the new template is available.

```bash
./charm-create.py -h
```
You should see python-pytest is listed as an installed template.
```bash
optional arguments:
  -h, --help            show this help message and exit
  -t TEMPLATE, --template TEMPLATE
                        Name of charm template to use; default is reactive-
                        python. Installed templates: python-pytest

```
# Template python-pytest
## Creating a charm
To create a new charm from the template call charm create and specify the
template as python-pytest. Additionally, charm-create accepts a charm name and
an optional root folder. My preference is to always provide both, if the
application is available in apt some metadata will be filled out for you. For
this example I'll create an empty charm for ddclient.

```bash
./charm-create.py -t python-pytest ddclient ../layer-ddclient
cd ../layer-ddclient
```

## Testing
Before going through how the template is laid out we'll kick off a full set of
tests. If you are familiar with charming and python tests this might be all the
demo you need to get started. Deploying the charm can take some time so this
also allows tests to run while you read about the template.

Since I'm doing this in a fresh bionic LXD I'll setup juju for local deploys,
you can [read about that here][juju-lxd] if you are new to juju. I'm running as
the user 'ubuntu' on bionic in a LXD container which can be setup with:

```bash
sudo snap install juju --classic
juju bootstrap localhost lxd
```
If everything went well `juju status` should show an empty model named default
is ready for use. Now you can run the tests with from the root of the template
folder you'll and will need just two dependencies make and tox:

```bash
sudo apt install make tox
JUJU_REPOSITORY=$(pwd) make test
```
In this case I'm setting the current directory to the JUJU_REPOSITORY because
I'm running in a container. I have an environment variable set for a common
location that I build charms to on my workstation which removes the need to
specify the repository location when calling make. If you are new to charming you can [read
more][charm-docs] in the getting started documents about the environment
variables. However, with this template only the JUJU_REPOSITORY need to be set
because the Layer and Interface paths are set by the template.

That's all you should need. This will install a python virtual environment for
unit testing, unit test the template, build it into a charm, install a virtual
environment for  functional testing, and deploy the charm with libjuju. You'll see reports for 
the tests that pass, unit test coverage, and the result of linting with flake8.

The output looks like this: 
```bash
$ JUJU_REPOSITORY=$(pwd) make test
unit installed: atomicwrites==1.2.1,attrs==18.2.0,charmhelpers==0.19.7,charms.reactive==1.1.2,coverage==4.5.2,Jinja2==2.10,MarkupSafe==1.1.0,mock==2.0.0,more-itertools==5.0.0,netaddr==0.7.19,pbr==5.1.1,pkg-resources==0.0.0,pluggy==0.8.1,py==1.7.0,pyaml==18.11.0,pytest==4.1.1,pytest-cov==2.6.1,PyYAML==3.13,six==1.12.0,Tempita==0.5.2
unit runtests: PYTHONHASHSEED='3584756322'
unit runtests: commands[0] | pytest -v --ignore /home/ubuntu/charm-template/layer-ddclient/src/tests/functional --cov=lib --cov=reactice --cov=actions --cov-report=term
========================================= test session starts ==========================================
platform linux -- Python 3.6.7, pytest-4.1.1, py-1.7.0, pluggy-0.8.1 -- /home/ubuntu/charm-template/layer-ddclient/src/.tox/unit/bin/python3
cachedir: .pytest_cache
rootdir: /home/ubuntu/charm-template/layer-ddclient/src, inifile:
plugins: cov-2.6.1
collected 3 items                                                                                      

tests/unit/test_actions.py::TestActions::test_example_action PASSED                              [ 33%]
tests/unit/test_lib.py::TestLib::test_pytest PASSED                                              [ 66%]
tests/unit/test_lib.py::TestLib::test_ddclient PASSED                                            [100%]Coverage.py warning: Module reactice was never imported. (module-not-imported)


----------- coverage: platform linux, python 3.6.7-final-0 -----------
Name                     Stmts   Miss  Cover
--------------------------------------------
actions/example-action       3      0   100%
lib/lib_ddclient.py          6      1    83%
--------------------------------------------
TOTAL                        9      1    89%


======================================= 3 passed in 0.40 seconds =======================================
_______________________________________________ summary ________________________________________________
  unit: commands succeeded
  congratulations :)
Building charm to base directory /home/ubuntu/charm-template/layer-ddclient
fatal: No names found, cannot describe anything.
Makefile:36: recipe for target 'build' failed
make: [build] Error 128 (ignored)
build: Destination charm directory: /home/ubuntu/charm-template/layer-ddclient/builds/ddclient
build: Please add a `repo` key to your layer.yaml, with a url from which your layer can be cloned.
build: Processing layer: layer:options
build: Processing layer: layer:basic
build: Processing layer: ddclient (from src)
proof: I: `display-name` not provided, add for custom naming in the UI
proof: W: Includes template README.ex file
proof: W: README.ex includes boilerplate: Step by step instructions on using the charm:
proof: W: README.ex includes boilerplate: You can then browse to http://ip-address to configure the service.
proof: W: README.ex includes boilerplate: - Upstream mailing list or contact information
proof: W: README.ex includes boilerplate: - Feel free to add things if it\'s useful for users
proof: I: all charms should provide at least one thing
functional installed: asn1crypto==0.24.0,atomicwrites==1.2.1,attrs==18.2.0,bcrypt==3.1.6,certifi==2018.11.29,cffi==1.11.5,chardet==3.0.4,cryptography==2.5,flake8==3.6.0,idna==2.8,juju==0.11.2,jujubundlelib==0.5.6,macaroonbakery==1.2.1,mccabe==0.6.1,mock==2.0.0,more-itertools==5.0.0,paramiko==2.4.2,pbr==5.1.1,pkg-resources==0.0.0,pluggy==0.8.1,protobuf==3.6.1,py==1.7.0,pyasn1==0.4.5,pycodestyle==2.4.0,pycparser==2.19,pyflakes==2.0.0,pymacaroons==0.13.0,PyNaCl==1.3.0,pyRFC3339==1.1,pytest==4.1.1,pytest-asyncio==0.10.0,pytz==2018.9,PyYAML==3.13,requests==2.21.0,six==1.12.0,theblues==0.5.1,urllib3==1.24.1,websockets==7.0
functional runtests: PYTHONHASHSEED='3922428688'
functional runtests: commands[0] | pytest -v --ignore /home/ubuntu/charm-template/layer-ddclient/src/tests/unit
========================================= test session starts ==========================================
platform linux -- Python 3.6.7, pytest-4.1.1, py-1.7.0, pluggy-0.8.1 -- /home/ubuntu/charm-template/layer-ddclient/src/.tox/functional/bin/python3
cachedir: .pytest_cache
rootdir: /home/ubuntu/charm-template/layer-ddclient/src, inifile:
plugins: asyncio-0.10.0
collected 4 items                                                                                      

tests/functional/test_deploy.py::test_ddclient_deploy[xenial] PASSED                             [ 25%]
tests/functional/test_deploy.py::test_ddclient_deploy[bionic] PASSED                             [ 50%]
tests/functional/test_deploy.py::test_ddclient_status PASSED                                     [ 75%]
tests/functional/test_deploy.py::test_example_action PASSED                                      [100%]

====================================== 4 passed in 353.29 seconds ======================================
_______________________________________________ summary ________________________________________________
  functional: commands succeeded
  congratulations :)
Running flake8
lint installed: flake8==3.6.0,mccabe==0.6.1,pkg-resources==0.0.0,pycodestyle==2.4.0,pyflakes==2.0.0
lint runtests: PYTHONHASHSEED='1094685518'
lint runtests: commands[0] | flake8
_______________________________________________ summary ________________________________________________
  lint: commands succeeded
  congratulations :)
```

You can run `juju status` to see the application has deployed for both xenial
and bionic.

```bash
$ juju status
Model    Controller  Cloud/Region         Version  SLA          Timestamp
default  lxd         localhost/localhost  2.5.0    unsupported  15:49:04Z

App              Version  Status  Scale  Charm     Store  Rev  OS      Notes
ddclient-bionic           active      1  ddclient  local    0  ubuntu  
ddclient-xenial           active      1  ddclient  local    0  ubuntu  

Unit                Workload  Agent  Machine  Public address  Ports  Message
ddclient-bionic/0*  active    idle   1        10.229.130.66          
ddclient-xenial/0*  active    idle   0        10.229.130.207         

Machine  State    DNS             Inst id        Series  AZ  Message
0        started  10.229.130.207  juju-3f9a32-0  xenial      Running
1        started  10.229.130.66   juju-3f9a32-1  bionic      Running

```

At this point you have an empty but deployable charm which unit tests and
deploys locally. If you are familiar with charming, this might be all you
need to get started. If you are new to charming you can use the template to
learn even before you understand all of the moving parts. Eventually you will
want to understand how testing is done. I'll follow up with additional post(s) 
to go through the details of how the template is organized and why.

To clean up your model you can destroy the default model and create a clean one
to deploy to again.
```bash
juju destroy-model default && juju add-model default
```
Be careful when using destroy-model, it will remove all machines in the model. I
do not install my 'production' workloads in the model named default. I consider
anything in such a model expendable. I recommend getting in a similar habit to
avoid completely removing a production model. While you will be prompted, I
destroy and recreate the default models on my local host frequently enough it's
muscle memory at this point and a warning prompt no longer causes me any pause.
Pick a name for testing and stick to it, it just might save your deployment.

[charm-template]: https://github.com/pirate-charmers/charm-template.git 
[juju-lxd]: https://docs.jujucharms.com/2.5/en/clouds-LXD
[container-blog]: https://blog.ubuntu.com/2015/10/30/nested-containers-in-lxd
[charm-docs]: https://docs.jujucharms.com/2.4/en/developer-getting-started
[template-repo]: https://github.com/pirate-charmers/template-python-pytest
