# Prerequisites

## Libvirt Platform

This tutorial leverages libvirt and KVM/QEMU to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. First step is to find a baremetal server with enough resources to run a Kubernetes cluster on virtual machines. During this tutorial I am about to configure a high availability Kubernetes cluster made with:

- 3 Master nodes (16GB RAM, 4vCPUs)
- 3 Worker nodes (16GB, 4vCPUs)
- 1 Loadbalancer which in this case will be based in Haproxy (2GB RAM, 2vCPUs)
- Operating system of choice: CentOS 7

In my case I have the luck to be allowed to use a Dell Blade with the following resources

```
Host: smc-master
Server model: PowerEdge M630	
Cpus: 2 x Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz	
Memory: 128.0 GB	
```

In case you are not as lucky as me, you can still follow this tutorial with the resources you have. Basically you can slow down the resources needed for each VM or use one worker instead of 3. I suggest to run 3 masters since this is the minimal number of masters to deploy a high available Kubernetes cluster.

Then, we can start configuring the baremetal server. Note that these steps are required to be executed by an administrator of the physical server (smc-master)

First, install all the virtualization packages needed to create virtual machines (VMs), virtual networks, disks and all virtual devices needed to run the provision the cluster.

```
yum groupinstall "Virtualization Host"

```

Also I suggest tp install the libvirtd client in the baremetal server itself in case we need to troubleshoot locally any issue that can arise.

```
yum install libvirt-client
```

Finally, start and enable the systemd service:

```
[root@smc-master ~]# systemctl enable libvirtd --now
```

## Libvirt CLI

In order to deploy all the virtual devices needed to run the infrastructure we can make use of the virsh command line. The virsh program is the main interface for managing virsh guest domains. The program can be used to create, pause, and shutdown domains. It can also be used to list current domains.

However, I find much easier to use [kcli])(https://kcli.readthedocs.io/en/master/) to deploy my virtual infrastructure. Kcli is a tool meant to interact with existing virtualization providers (libvirt, kubevirt, ovirt, openstack, gcp and aws, vsphere) and to easily deploy and customize vms from cloud images.  You can also interact with those vms (list, info, ssh, start, stop, delete, console, serialconsole, add/delete disk, add/delete nic,…). Futhermore, you can deploy vms using predefined profiles, several at once using plan files or entire products for which plans were already created for you.


### Install kcli

Follow the kcli [documentation](https://kcli.readthedocs.io/en/master/#installation) to install and configure all the binaries needed to manage the baremetal server

If using CentOS, which is our case

```
yum install epel-release
```

```
yum install pip3 python3 libvirt-python libvirt-devel gcc
```

```
pip3 install kcli
```

In case you do not want to install kcli on your laptop or remote system, you can use kcli's container image stored in docker hub:

```
docker pull karmab/kcli
docker run --rm karmab/kcli
```

```
alias kcli='docker run --net host -it --rm --security-opt label=disable -v $HOME/.ssh:/root/.ssh -v $HOME/.kcli:/root/.kcli -v /var/lib/libvirt/images:/var/lib/libvirt/images -v /var/run/libvirt:/var/run/libvirt -v $PWD:/workdir -v /var/tmp:/ignitiondir karmab/kcli'

alias kclishell='docker run --net host -it --rm --security-opt label=disable -v $HOME/.ssh:/root/.ssh -v $HOME/.kcli:/root/.kcli -v /var/lib/libvirt/images:/var/lib/libvirt/images -v /var/run/libvirt:/var/run/libvirt -v $PWD:/workdir -v /var/tmp:/ignitiondir --entrypoint=/bin/sh karmab/kcli'
```

### Configure kcli to manage the baremetal server 

Once you have kcli configured in your baremetal server you need to create a ssh key pair to interact with the libvirt daemon.

```
[root@smc-master ~]# ssh-keygen -t rsa -b 2048
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:C4/ecwZji6RNyQ91q9UVs7P0HF4Jj/u0mgcco3A6jVs root@smc-master.cloud.lab.eng.bos.redhat.com
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|             .o  |
|              ++.|
|        .... +=+.|
|     ..oS.*ooo==o|
|      *++=oE.+.oo|
|     =.=o*+   + .|
|    ..o.=.o   .+ |
|      . .+   oo  |
+----[SHA256]-----+
```

Establish the connection between the kcli and the libvirtd. In this case we have named the libvirt baremetal server as 8s-hard-way-libvirt
```
[root@smc-master ~]# kcli create host kvm -H 127.0.0.1 k8s-hard-way-libvirt

```


```
$ vim ~/.kcli/config.yml

default:
  client: smc-master
  cloudinit: true
  enableroot: true
  insecure: true
  nested: true
  reservedns: false
  reservehost: false
  reserveip: false
  start: true
  tunnel: false
  pool: default
local:
  nets:
  - default

smc-master:
  type: kvm
  host: 10.19.138.41
  protocol: ssh
  user: root
  tunnel: true
  pool: default
```

Create the pool where the images and vms are going to be placed in the baremetal server

```
[root@smc-master ~]# kcli create pool -p /var/lib/libvirt/images/ default
Adding pool default...
```


Test if working by trying to gather the instances running:

```
openstack server list[root@smc-master ~]# kcli info host
Connection: qemu:///system
Host: smc-master.cloud.lab.eng.bos.redhat.com
Cpus: 32
Vms Running: 0
Memory Used: 0MB of 130850MB
Storage:default Type: dir Path:/var/lib/libvirt/images Used space: 2.52GB Available space: 550.83GB
Network: em1 Type: bridged
Network: em2 Type: bridged
Network: em3 Type: bridged
Network: em4 Type: bridged
Network: p2p1 Type: bridged
Network: p2p2 Type: bridged
Network: p2p3 Type: bridged
Network: p2p4 Type: bridged
Network: default Type: routed Cidr: 192.168.122.0/24 Dhcp: True
```

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` followed by `shift+:`. Next type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
