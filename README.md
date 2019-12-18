# minishift-setup
This repository describes a minishift local setup using virtualbox and helm
for local development of application deployed to OpenShift.

## Warning

Testing is done on Debian Linux and there are no guarantees that it will work anywhere
else out of the box even when it should. Because we will use Virtualbox 
(open source version is ok) as virtualization abstraction layer which will wrap 
around (kvm, hyperv, and the mac thingy),
 it should not play a role weather you install on linux, windows or mac. In theory...
 
## Todos for this guide
- [ ] describe minishift installation process 
 
## Prerequisites

1. Virtualbox 6+ installed
2. Working (fast) internet connection

## Setup
#### Download minishift and install release

> I used verseion 1.34.2 but go ahead and always try the newest one

https://github.com/minishift/minishift/releases

Install it on your system and put it on your PATH so you can actually
 run this from the command line

```shell script
minishift version
```

#### Create minishift virtual machine

>###### Little word on network connectivity
>1. Your PC should have internet (>10Mbit) or the installation will be slow as hell
>2. Your PC IP address probably comes from from a DHCP server and that fine, we do not need to change that
>3. **Verify** that you can ping 8.8.8.8 or minishift will not boot
>4. **Turn OFF** your firewall because it will mess with minishift
>5. **Accept that** machines deployed in the minishift **will NOT** be accessible from other computers _(This might actually change)_

>In case you already have an installation which does not fit your needs and are OK with replacing it with this
>than do a complete wipeout like this
>```shell script
>minishift delete --force --clear-cache
>rm -rf ~/.kube
>rm -rf ~/.helm/
>rm -rf ~/.minishift/
>```
>This will **WIPE EVERYTHING** from your previous minishift installation and may also mess UP with helm
>and kubernetes setups so take special care!

Now enable all default addons on minishift and enable admin access

```shell script
minishift addons install --defaults && echo OK
minishift addons enable admin-user && echo OK
```

>Every command should be terminated with 'OK'. Sometimes, when an error happens while executing the minishift
>command, it actually does not look like an actual error on the first sight and is easily overlooked.
 
 




      
