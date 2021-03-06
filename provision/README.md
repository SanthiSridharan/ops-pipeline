
# How to rebuild the opspipeline VMs

There are 2 main ways to go about building more future opspipeline revisions.

1. Run builds manually
1. Stand up a local VMware vm with tools to build the VMs and manage
   your own vagrant version catalog
1. Leverage [Atlas](https://atlas.hashicorp.com/) to build and host boxes


## Building manually

This is really a subset of the following section.  If you want to run builds manually,
install:

* [packer](http://packer.io)
* vmware, virtualbox, docker, as needed for what you want to build
* chefdk

Then you can run builds as described in the [section below](#build).


## Build your own build server for building VMs

This requires that you have a VMware hypervisor.  Virtualbox doesn't fully
support nested vms.


### installing needed packages and vagrant plugins
You may need to install some of the following vagrant plugins to get everything working:


    # needed for chef provisioning
    vagrant plugin install vagrant-berkshelf
    brew cask install chefdk # needs to be in sync with the vagrant plugin version

    # one of the following based on provider
    vagrant plugin install vagrant-vmware-fusion
    vagrant plugin install vagrant-vmware-workstation
    vagrant plugin install vagrant-vmware-appcatalyst

    # for windows
    choco install chefdk
    choco install packer
    vagrant plugin install berkshelf-vagrant
    vagrant plugin install vagrant-vmware-workstation
    vagrant plugin license vagrant-vmware-workstation \Users\....\license.lic
    # also, be sure to run any local packer commands in an admin shell



*Note, you'll need to manually set VMware Workstation to use a trial license,
 or set a real license number in the jenkins-master attributes.  Get into
 the desktop, and run Workstation once to set the license.  Alternativly, you can
 set one in the provisioning, just be careful not to let a license get committed.*

1. Stand up a VM with tools to package virtualbox and VMware vms

        $ # stand up a build server
        $ git clone git@github.com:capitalone/opspipeline.git
        $ cp Vagrantfiles/builder/Vagrantfile .
        $ # the provider could also be vmware_workstation or vmware_appcatalyst for your OS
        $ vagrant up --provider=vmware_fusion
        $ berks vendor provision/chef/vendor-cookbooks


1. <a name="build"/>Log in and start a build

   Builds are done as 2 stages to allow saving some time by reusing
   the same base VM as a starting point for provisioning.

        vagrant ssh
        cd /vagrant
        version=0.20

        # first stage builds the vms for both vmware and virtualbox, which can be
        # re-used in the next stage

        # first an ubuntu base
        packer build -var "version=$version" packer/ubuntu-iso-to-vm.json

        # then a centOS base
        packer build -var "version=$version" packer/centos-iso-to-vm.json

        # second stage packages the boxes by provisioning on top of the base vm
        packer build -var "version=$version" packer/headless-stage-2.json

        # later ....
        ls *.box

        # Add the boxes to your local setup.
        # Alternativly, upload them to a url internally (discussed below), or atlas.
        vagrant box add --force --name opspipeline/headless headless-vmware-iso-${version}.box
        vagrant box add --force --name opspipeline/headless headless-virtualbox-iso-${version}.box

## Leverage Atlas

### Atlas high level workflow
The builds intended for Atlas are at `packer/headless.json` and `packer/desktop.json`.
Running `packer push` with those will run a buils remotely on atlas.

### Actual steps for pushing to Atlas

1. Get an account at [Atlas](https://atlas.hashicorp.com/)

1. Create and save an api token (under settings)

1. [install packer](http://www.packer.io/intro/getting-started/setup.html)

1. clone source

        git clone git@github.com:capitalone/opspipeline.git

1. copy all chef recipes into place for shipping to atlas

        berks vendor provision/chef/vendor-cookbooks

1. push the build to Atlas

        export ATLAS_TOKEN=9NS.....xxxxx
        packer push packer/headless.json
        # this returns after uploading the local tree to atlas
        # check Atlas to see when build is complete

## setting up an internal box repo:

You could use Artifactory for hosting an internal vagrant repo, or setup a web server
to serve the following files and boxes at the correct urls.

Write a catalog.json file similar to this.  Update version numbers, and checksums.

      "name": "opspipeline",
      "description": "opspipeline, based on ubuntu-14.04 LTS",
      "versions": [
          {
              "version": "0.1",
              "providers": [
                  {
                      "name": "virtualbox",
                      "url": "http://reposerver.local/opspipeline/headless-virtualbox-0.1.box",
                      "checksum_type": "sha1",
                      "checksum": "46841c18ebd41344cfb28409b2a9c5ff43ae1bf2"
                  },
                  {
                      "name": "vmware_desktop",
                      "url": "http://reposerver.local/opspipeline/headless-vmware-0.1.box",
                      "checksum_type": "sha1",
                      "checksum": "960d0dfe6fdb38a3a96f68128b2a01c096f6dc85"
                  }
              ]
          },
          {
              "version": "0.2",
              "providers": [
                  {
                      "name": "virtualbox",
                      "url": "http://reposerver.local/opspipeline/headless-virtualbox-0.2.box",
                      "checksum_type": "sha1",
                      "checksum": "88122aa0dd3494cef7379d97d92d36d2d4d14ab1"
                  },
                  {
                      "name": "vmware_desktop",
                      "url": "http://reposerver.local/opspipeline/headless-vmware-0.2.box",
                      "checksum_type": "sha1",
                      "checksum": "a470e3ff5e30714cd8f5b9ffc0e5304d273675d1"
                  }
              ]
          },
          {
              "version": "0.4",
              "providers": [
                  {
                      "name": "virtualbox",
                      "url": "http://reposerver.local/opspipeline/headless-virtualbox-iso-0.4.box",
                      "checksum_type": "sha1",
                      "checksum": "ed130d4ca164b42db3575b439f361fbda3e9fa26"
                  },
                  {
                      "name": "vmware_desktop",
                      "url": "http://reposerver.local/opspipeline/headless-vmware-iso-0.4.box",
                      "checksum_type": "sha1",
                      "checksum": "c0ec071e38ad5988a5ce502125c65c01be675379"
                  }
              ]
          }
      ]
  }

In your `Vagrantfile`, add the following:

    config.vm.box = "opspipeline/headless"
    config.vm.box_url = "http://reposerver.local/opspipeline/catalog.json"

Now, initial vagrant up will download the latest version in the catalog.  You can also destroy
and re-up with the latest version.


## Testing chef-solo with test kitchen

Install (Test Kitchen)[https://github.com/test-kitchen/test-kitchen]

If you are updating chef recipies, you'll want to write a test first,
then run the following for the platforms you want to test against.  You
should get passing tets before committing the changes.

    kitchen converge virtualbox
    kitchen verify virtualbox

This iterates through upping, provisioning, and testing the headless and desktop
opspipeline boxes for virtualbox provider.  The following commands do the same for the
vmware provider.

    kitchen converge vmware
    kitchen verify vmware

## Troubleshooting build problems

#### `==> vmware-iso: Error starting VM: VMware error: Error: The operation was canceled`

That error may show up if your build vm doesn't have enough ram to bring up the contained vmware vm.

#### for verbose logging, run with

    PACKER_LOG=1  PACKER_LOG_PATH=./debug.log packer build headless-stage-1.json
    PACKER_LOG=1  PACKER_LOG_PATH=./debug.log packer build headless-stage-2.json

#### if you can't figure out the error, look for logs from the vm provider.

Start a build on command line, use `^z` to send it into the background.  Look for
logfiles undoer the output-* directories.  That will contain errors directly from
the vm provider.  They will get cleaned up when the build finishes well or failed,
so you'll need to see them before the build ends.
