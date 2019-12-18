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
- [x] describe minishift installation process 
- [ ] describe helm installation process
 
## Prerequisites

1. Virtualbox 6+ installed
2. Working (fast) internet connection

## Setup
#### 1. Download minishift and install release

> I used verseion 1.34.2 but go ahead and always try the newest one

https://github.com/minishift/minishift/releases

Install it on your system and put it on your PATH so you can actually
 run this from the command line

```shell script
minishift version
```

#### 2. Create minishift virtual machine

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
minishift addons install --defaults
minishift addons enable admin-user
```

Now lets create (provision) minishift instance

```shell script
minishift start --profile myminishiftV1 --vm-driver virtualbox --memory 8GB --cpus=4 --disk-size=50GB --openshift-version=v3.11.0 --host-only-cidr=10.11.12.1/24
```

>Explanation of the less obvious parameters:
>
>--profile - name for the minishift instance (you can have more installed in parallel); choose whatever fits you
>
>--host-only-cidr - is the ip/network for the network interface connecting your PC with the minishift;
> make here a reasonable choice which does not conflict with your networks  

If it worked than you should get something like this and drop back to a shell:
```
Login to server ...
Creating initial project "myproject" ...
Server Information ...
OpenShift server started.

The server is accessible via web console at:
    https://10.11.12.100:8443/console

You are logged in as:
    User:     developer
    Password: <any value>

To login as administrator:
    oc login -u system:admin


-- Exporting of OpenShift images is occuring in background process with pid 3736.
```
**Please copy that section and store it somewhere because you will need the information later!!!** 

As you can see, the last sentence says that there is still something happening in the background with a 
process ID of 3736. Open some process monitoring and see if you can find that process and if it is using
a good portion of the CPU. That is openshift *actually* booting so be patient and wait until it finishes
its work. It usually takes couple of minutes.

#### 3. Stop minishift and create a snapshot of clean installation

Minishift is up and running and before we proceed to next steps which are definitely more complicated,
we want to save the current state if form of a Virtualbox snapshot in case something goes wrong, so we do 
not have to redo the whole setup again.

Now shutdown minishift
```shell script
minishift stop
```

> (Optional but recomended) VirtualBOX tuning for better performance:
>```shell script
>VBoxManage modifyvm myminishiftV1 --nictype1 virtio
>VBoxManage modifyvm myminishiftV1 --nictype2 virtio
>```

Now take a snapshot of the virtual machine so we can revert back to it when we experience problems

```shell script
VBoxManage snapshot myminishiftV1 take clean_install
``` 

#### 3. Install helm and tiller

First we need to start minishift again.
```shell script
minishift start
```

>root login to minishift (for advanced users)
>```shell script
>minishift ssh
>``` 
>login: docker
> pass: tcuser
>
>switch to root:
>```shell script
>sudo -s
>```




      
