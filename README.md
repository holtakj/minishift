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
- [x] describe helm 2.x installation process
- [x] setup docker repositories
- [ ] describe how to create and remove a test setup - application deployed by helm
- [ ] Upgrade to helm 3.x
 
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

>root login to minishift (for advanced users) execute: (or use virtualbox console)
>```shell script
>minishift ssh
>``` 
>login: docker
>
>pass: tcuser
>
>switch to root:
>```shell script
>sudo -s
>```

First step is to setup OC which is the client side to manage openshift on your PC.


Now you have to install helm on your PC:

https://github.com/helm/helm/releases

Download und unpack the latest 2.x version as version 3.x is not tested until now.
Put it on your PATH.

Run this and it should show the client version and complain about acess to "pods" which is fine as we
did not install the server part until now

```shell script
helm version
```

Now lets install minishift client environment:

Run this and it will tell you what to do:
```shell script
minishift oc-env
```

On my system it says to change PATH to this:
```shell script
export PATH="$HOME/.minishift/cache/oc/v3.11.0/linux:$PATH"
```

Run this to verify that the setup works:
```shell script
oc status
```

Now apply these commands to install Tiller which is the server part of helm into minishift.
Usually tiller is installed directly into kubernetes namespace on minishift but this causes problems
with openshift versions greater than 3.9.0 so we will create an own namespace for tiller and give it
 enough rights to do its job.

```shell script
# first start with activating the admin user addon (again) 
minishift addon apply admin-user

# now install tiller into its own namespace as superuser
oc login -u admin -p admin
oc new-project tiller-namespace

# create a system account for tiller
oc create sa tiller
# in a real cluster I wouldn't grant Tiller cluster admin but this is way more easier to set up
oc adm policy add-cluster-role-to-user cluster-admin -z tiller
# the default developer user must be able to "see" tiller from outside of minishift
oc policy add-role-to-user view developer -n tiller-namespace
oc create role podreader --verb=get,list,watch --resource=pod -n tiller-namespace
oc adm policy add-role-to-user podreader developer --role-namespace=tiller-namespace -n tiller-namespace
# our regular developer user must be able to communicate with tiller from outside of minishift
oc create role portforward --verb=create,get,list,watch --resource=pods/portforward -n tiller-namespace
oc adm policy add-role-to-user portforward developer --role-namespace=tiller-namespace -n tiller-namespace
# our regular develpoer should see that the roles are installed when running the wizard to not get warnings
oc create role rolereader --verb=get,list,watch --resource=roles -n tiller-namespace
oc adm policy add-role-to-user rolereader developer --role-namespace=tiller-namespace -n tiller-namespace
# install tiller
helm init --service-account tiller --tiller-namespace tiller-namespace
# expose tiller port to the host computer so you can run helm from your PC
oc expose deployment/tiller-deploy --target-port=tiller --type=NodePort --name=tiller -n tiller-namespace
```

And we are done. Now lets verify some things because they are critical.

Run this and it should give you a json with the tiller service configuration. Look for 'nodePort' parameter.
If it is not present or empty, you will not be able to make a connection to tiller.

```shell script
oc get svc/tiller -o json
```

#### 4. Setup environment

You need to set some environment variables in order to work with Helm:

```shell script
MINISHIFT_VERSION=`minishift version`

echo Initialising Minishift environment $MINISHIFT_VERSION
eval $(minishift oc-env)

export TILLER_NAMESPACE=tiller-namespace 
echo TILLER_NAMESPACE=$TILLER_NAMESPACE

export HELM_HOST="$(minishift ip):$(oc get svc/tiller -o jsonpath='{.spec.ports[0].nodePort}' -n $TILLER_NAMESPACE --as=system:admin)"
echo HELM_HOST=$HELM_HOST

export TILLER_HOST=$HELM_HOST
echo TILLER_HOST=$TILLER_HOST

export MINISHIFT_ADMIN_CONTEXT="default/$(oc config view -o jsonpath='{.contexts[?(@.name=="minishift")].context.cluster}')/system:admin"
echo MINISHIFT_ADMIN_CONTEXT=$MINISHIFT_ADMIN_CONTEXT

export KUBE_CONTEXT="default/$(oc config view -o jsonpath='{.contexts[?(@.name=="minishift")].context.cluster}')/system:admin"
echo KUBE_CONTEXT=$KUBE_CONTEXT
```

which gives me this output

```
Initialising Minishift environment minishift v1.34.2+83ebaab
TILLER_NAMESPACE=tiller-namespace
HELM_HOST=10.11.12.100:31172
TILLER_HOST=10.11.12.100:31172
MINISHIFT_ADMIN_CONTEXT=default//system:admin
KUBE_CONTEXT=default//system:admin
```

and now I can try to check if helm works by getting the versions of client and server part
```shell script
helm version
```

should give you
```
Client: &version.Version{SemVer:"v2.16.1", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.16.1", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}
```

If the command hangs, you probably did not set HELM_HOST variable properly.

Now it is time to shutdown and create a new snapshot

```shell script
minishift stop
VBoxManage snapshot myminishiftV1 take helm_install
```

#### 5. Docker registry

You need a docker registry where you can deploy your images to. For development, you can create a docker registry
on your pc and use that. How you create a registry is up to you but under Debian it is very simple.

```shell script
sudo apt install docker-registry
```

or more generic way which works on every OS but you wont be getting security updates automatically

```shell script
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

The docker registry is HTTP only and listens on port 5000 by default. You have to reflect this when yoy use it.

Next thing is to create a hostname for your docker registry on a IP address which can be seen from minishift.
Remember when you used the --host-only-cidr=10.11.12.1/24 parameter while initializing minishift? This is your IP.
Try to ping it.

```shell script
ping 10.11.12.1
```

If it work than all you have to do is to enter following into your hosts-file.
```
10.11.12.1 registry.local
```

Now tell your local docker daemon about the registry. Usually you have to configure a insecure registry.
It os littlebit OS dependent how you do this. Under debian, you would change the file /etc/docker/daemon.json
to look like this and restart docker for it to take effect.

```json
{
    "max-concurrent-uploads" : 1,
    "insecure-registries" : ["registry.local:5000"]
}
```

```shell script
sudo systemctl restart docker.service
```

Now push your image into that registry. It could look like this. I hate the way docker is doing it.
```shell script
docker tag myimage:latest registry.local/myimage:latest
docker push registry.local/myimage:latest
```

If it worked, great! Last step is to tell minishift about your new repository. That is fairly easy:

```shell script
minishift config set insecure-registry registry.local:5000
```     
      
You are ready to go!

Push images into your local repository and deploy them on minishift!
