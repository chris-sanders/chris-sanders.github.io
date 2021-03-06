---
layout: post
title:  "Pytest Template"
subtitle: "Design details"
date:   2019-02-07 20:00:00 +0000
gh-repo: pirate-charmers/template-python-pytest
gh-badge: [star, fork, follow]
share-img: "/img/design-blueprint.png"
image: "/img/design-blueprint.png"
tags: [charms, pirate-charmers, charm-template]
---
This post builds on [a previous post][part1] to go into details of the
python-pytest charm template. In this post I will review the layout of the charm
and how to write/expand the testing for your charm. Writing code with testing in
mind from the start can make testing significantly easier.

# Template layout
This post continues with examples from [the first post in the series][part1] and
will assume you already have the example charm from that post setup.

## Root Folder
In the root folder of your charm you will have:
 - Makefile
 - interfaces
 - layers
 - src

The Makefile drives the build/test process, more on that later. 

The `src` folder containers the actual charm code, and the `interfaces` and
`layers` folders are intended to pin revisions of any interfaces and layers you
use in the charm. These two folders are empty on a new template. The build
process will look for interfaces and layers in these folders when building the
charm in the src folder. This allows the use of [git submodules][submodules] to pin layers and
interfaces to specific branches and revisions. If you do not use this the build
process will pull the latest version of these included interfaces/layers every
time you build leaving the build ambiguous. 

Submodules will be used for most
interfaces/layers we use in pirate-charmers as we believe *Explicit is better
than implicit* and we don't like builds that break/change each build with no
change to the source repository. It is however your choice, the template does
not force the use of submodules.

There is one other benefit of using submodules, you do not have to use
interfaces or layers that are registered in the [layer-index][layer-index] since
they are stored locally. While I encourage you to submit layers and interfaces
that you've written to the index, there are sure to be times that you have
something that isn't a good fit for a global index. For example, when working on
a new interface or expanding an interface that isn't yet stable enough to share
widely.

### Submodule example
As an example we'll add the [layer-version][layer-version] into the previously
created template. This layer creates a version file from a git tag at build time
and uses that as the 'Version' field in the charm. It's not in the index because the
version field is intended for *software version* not charm version. However,
until charm version is supported as a field I'm co-opting it in my charms as I
find the charm version more relevant than the software version since I can
derive the latter from the former.

To create the submodule you need to make the charm folder a git repository and
then add the submodule. If you've already built the charm, I recommend removing the folders
`builds` and `deps` before you do this, they are built each time and do not need
to be revisioned. They are in the charm folder because we are using the root folder
as the build path as previously mentioned.

```bash
rm -fr ./builds
rm -fr ./deps
git init
git add .
git commit -m "Initial commit"
cd ./layers
git submodule add https://github.com/pirate-charmers/layer-version.git
```
We'll also set up the layer so you can see that it worked. Edit the file
`src/layers.yaml` to add the following

```diff
includes:
  - layer:basic
+  - layer:version
+options:
+  version:
+    file_name: "repo-info"
```
Since the layer needs a git tag for the version add a tag after saving the
changes.

```bash
git add .
git commit -m "Added layer-version"
git tag 0.0.0
```
That's it, you now have a submodule tracking layer-version, which isn't in the
index, which will be compiled into the charm. Remember any submodules you
include will require explicitly updating in the future when/if you want to
include the updates in your charm.

## Building
Building the charm is driven by the Makefile. Before building, it's a good idea
to update any submodules with `make submodules`. You
can see the available options in the Makefile with `make help`.

To build the charm you can use `make release` which will build it to the
'builds' folder in the JUJU_REPOSITORY directory. Running the build command
will show the layers and where they are pulled from.

```bash
$ JUJU_REPOSITORY=$(pwd) make release
... redacted ...
build: Processing layer: layer:options
build: Processing layer: layer:basic
build: Processing layer: layer:version (from layers/layer-version)
build: Processing layer: ddclient (from src)
```
This snippet shows that layer options and layer basic are being
pulled from the index, layer version is from the layers folder, and the base
charm is from the src folder.

Additionally, if you run the functional tests you'll see the tag we added
(0.0.0) is now being used as the Version field.

```bash
App              Version  Status  Scale  Charm     Store  Rev  OS      Notes
ddclient-bionic  0.0.0    active      1  ddclient  local    0  ubuntu  
ddclient-xenial  0.0.0    active      1  ddclient  local    0  ubuntu 
```

This is being done because part of the build process is calling `git describe
--tags` and placing the output in the build folder as the file `repo-info`. If
you do not tag your repository you'll see an error during build, but the build
will continue and repo-info will be empty.

## Testing
The template is using [pytest][pytest] for unit and functional testing. If you
aren't familiar with it, you'll want to refer to the [fixture
documentation][fixture-docs] for details on fixtures. I have
found fixtures to provide a very clean way of scaling reusable test code in
charms and it's particularly important when dealing with async code in the
functional tests.

During all of the test runs (lint, unittest, functional) a virtual environment
will be created for dependencies. The first run will therefore take slightly
longer with additional runs reusing the environment. If you change the
dependencies you will likely need to run `make clean` to remove the environment
to force an install of the new dependencies. Alternately, I like to run `git clean -fdx` 
to remove all files not in git. Similarly if you run any of the tests from the
built folder you'll want to remove the .tox folder before uploading to the
charm store so the folder won't be included in every deploy of your charm.

### Unit Testing
The unit tests are called via [tox][tox], it runs tests found in the folder
`/src/tests/unit` and ignores functional tests found in `/src/tests/functional`.
The test folder has it's own requirements.txt file. This allows you to include 
packages specifically for unit test
separate from the charm level requirements file. Unfortunately, charm-tools
removes the root level requirements.txt file during charm building when it
fetches the requirements and bundles them in the charm. If you want unit tests
to run from the built charm folder, you'll need to include the base requirements
in the unit specific requirements file as well. I do want to be able to run unit tests
on the built charm so I always make the unit file a super-set of the one in the
charm root.

Not all parts of a charm are suitable for unit testing. Specifically the reactive
and hook portions of a charm are frequently of very little value at the unit
test stage. This template is setup with the intention of testing the charm logic
at the unit level and leaving the testing of hooks and subprocesses to
functional testing.

#### Code layout
The template sets up a class in the `src/lib` folder based on the charm name, which
is intended to hold charm logic for unit testing. By default it only loads the
charm config. With the example charm the file is `src/lib/lib_ddclient.py`.

To support the testing of this helper class there are several
[pytest-fixtures][fixture-docs] which pytest loads from the default file
`src/tests/unit/conftest.py`. Fixtures build on each other, and the last fixture
in the file named `ddclient` returns an instance of the class patched for many
of the common uses. 

```bash
@pytest.fixture
def ddclient(tmpdir, mock_hookenv_config, mock_charm_dir, monkeypatch):
    from lib_ddclient import DdclientHelper
    helper = DdclientHelper()

    # Example config file patching
    cfg_file = tmpdir.join('example.cfg')
    with open('./tests/unit/example.cfg', 'r') as src_file:
        cfg_file.write(src_file.read())
    helper.example_config_file = cfg_file.strpath

    # Any other functions that load helper will get this version
    monkeypatch.setattr('lib_ddclient.DdclientHelper', lambda: helper)

    return helper
```
The *ddclient* fixture is defined above using several other fixtures defined
earlier in the file. The *tmpdir* fixture is a pytest built in fixture and
generates files in `/tmp/pytest-*` folders as the name implies. This is a good
way to test the writing or manipulating of files without having to deploy the
charm. You can see where an known config file is written from the test directory
into the tmp directory and the path patched into the helper. Any methods in the
class that process this config file will have a valid known file to test
processing with.

With the fixture in place, writing a test that checks that the charm config has
been loaded is very straight forward. Here is the test from
`src/tests/unit/test_lib.py`

```bash
def test_ddclient(ddclient):
    ''' See if the helper fixture works to load charm configs '''
    assert isinstance(ddclient.charm_config, dict)
```
The above *ddclient* fixture is requested and the charm_config is verified to be
a dict as expected. For simple cases like this, the mocking can happen entirely
in the fixture and the test can focus on verifying that methods work as expected.

The use of pytest fixtures is beyond the scope of this post, but you can see
examples on the github including [layer-weechat][layer-weechat] and
[layer-haproxy][layer-haproxy]. Since the tests run out of the box, as you add
logic to the helper class you can test with `make unittest` for very quick
testing to get the logic working correctly before trying to deploy the charm and
do functional testing.

## Functional Tests
Functional testing is very similar to how unit testing is setup. Pytest is run
and fixtures are loaded from `src/tests/functional/conftest.py`. The template
does not have any fixtures or even a conftest.py file defined for functional
testing at this time. Instead a few basic fixtures are provided directly in the
test file before the tests.

Because the functional testing is driven by [libjuju][libjuju] which uses
asyncio you might want to [review the documenation][asyncio] if you are new to
this feature of python3.

By default the functional test will deploy the charm once for each series
specified in the test (xenial and bionic), check that the status
reaches 'active' on both units, and call the example action verifying the call
completes. Libjuju allows you to programmatically interact with juju deploying,
configuring, and even relating applications to each other. Fixtures are provided
for accessing the Model, Applications, and Units for the series defined at the
top of functional test file `src/tests/functional/test_deploy.py`.

```bash
@pytest.mark.parametrize('series', series)
async def test_ddclient_deploy(model, series):
    # Starts a deploy for each series
    await model.deploy('{}/builds/ddclient'.format(juju_repository),
                       series=series,
                       application_name='ddclient-{}'.format(series))
    assert True


async def test_ddclient_status(apps, model):
    # Verifies status for all deployed series of the charm
    for app in apps:
        await model.block_until(lambda: app.status == 'active')
    assert True
```
This snippet shows *test_ddclient_deploy* using the fixture *model* to start
deploying the charm once per series (xenial and bionic). After the deploys are 
successfully started
*test_ddclient_status* then waits for each application to reach the active
status, using the fixtures *model* and *apps*. All fixtures are defined at the
top of the *test_deploy.py* file above these tests.

Once the units are deployed actions can be run, files, and services checked, and
ports and APIs exercised to verify they are working as expected. Libjuju
provides several functions to inspect or run commands on units, and each unit
has a 'public_address' property that can be used to test web interfaces or API
calls. While using libjuju is charm specific using the requests library to check
web interfaces is not unique to charms or this template. As before you can
review the charms in the [pirate-charmers github][pirate-charmers] for examples
of ways we are testing functionally. As of this post, the most recent charm
which tests actions is [taskd][taskd-github]. Don't forget to include any python modules
you need in the functional test requirements.txt file to make them available in
the virtual environment during testing.

### Running functional tests
To run the functional tests you can run `make functional` from the root folder.
This will build the charm, and then run the tests. Building a charm can be time 
consuming, as can deploying it. Any
logic that can be reasonably tested in unit testing is best tested there with
functional testing verifying the configuration and states that a charm processes
which are difficult to unit test.

Some time can be saved while writing functional tests by skipping the charm
building step. If the charm has not changed, and you are working on writing the
tests in `src/tests/functional/test_deploy.py` you do not need to rebuild to
re-run the tests. You can do this by running the functional test from the
build folder, the tests will still run but the built charm will simply test the
currently built charm and not re-build it. If you do this be sure to 
copy your final test_deploy.py file back to your source folder or all changes
will be lost on the next build when the source folder is written back to the
build folder (overwriting any changes you made in the build folder).

You can save significant time while
writing tests by rerunning the tests without removing the model or applications
that were deployed. The charms will fail to deploy if they are already deployed
but the test suite will continue to run. This means if the deploy is passing and
you are writing test to exercise the units you do not need to remove and re-deploy 
them for each iteration.

## Closing
This post just scratches the surface of best practices for charm writting and
testing. After several years of charming, my understanding of those
best practice continues to evolve. This is a good start, and hopefully
helpful for anyone getting started that wants to start with what I have learned
during my time charming. However charms, like cloud software, are quickly
chaning and what I describe here is likely to continue to change. Please pay
attention to the post date. While I have every intention of continuing to update
the templates, I do not intend to update this post. It is a point in time guide
for writting better charms.

[part1]: https://chris-sanders.github.io/2019-01-28-charm-create/
[pytest]: https://docs.pytest.org/en/latest/
[fixture-docs]: https://docs.pytest.org/en/latest/fixture.html
[template-repo]: https://github.com/pirate-charmers/template-python-pytest
[submodules]: https://git-scm.com/book/en/v2/Git-Tools-Submodules
[layer-index]: https://github.com/juju/layer-index
[layer-version]: https://github.com/pirate-charmers/layer-version
[layer-weechat]: https://github.com/pirate-charmers/layer-weechat
[layer-haproxy]: https://github.com/pirate-charmers/layer-haproxy
[libjuju]: https://github.com/juju/python-libjuju
[asyncio]: https://docs.python.org/3/library/asyncio.html
[pirate-charmers]: https://github.com/pirate-charmers
[tox]: https://tox.readthedocs.io/en/latest/
[taskd-github]: https://github.com/pirate-charmers/layer-taskd
