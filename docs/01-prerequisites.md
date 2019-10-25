# Prerequisites

## Libvirt Platform

This tutorial leverages libvirt and KVM/QEMU to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. First step is to find a baremetal server with enough resources to run a Kubernetes cluster on virtual machines. In my case I am lucky to borrow a Dell Blade with the following resources

```
Host: smc-master
Server model: PowerEdge M630	
Cpus: 2 x Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz	
Memory: 128.0 GB	
```

During this tutorial I am about to configure a high availability Kubernetes cluster made by the following virtual machines. Note that all will run virtualized in the baremetal server:

   VM Name      | Purpose    |   OS     | vCPUs | Memory | Disk  |
| ------------- | ---------- | ---------|-------|--------|-------|---
| master00      | controller | CentOS 7 |   4   |  16 GB | 50 GB |
| master01      | controller | CentOS 7 |   4   |  16 GB | 50 GB |
| master02      | controller | CentOS 7 |   4   |  16 GB | 50 GB |
| worker00      | controller | CentOS 7 |   4   |  16 GB | 50 GB |
| worker01      | controller | CentOS 7 |   4   |  16 GB | 50 GB |
| worker02      | controller | CentOS 7 |   4   |  16 GB | 50 GB |
| loadbalancer  | balancer   | CentOS 7 |   2   |  2 GB  | 20 GB |

- Ansible (optional)

In case you are not as lucky as me, you can still follow this tutorial with the resources you have. Basically you can decrease the resources assigned to each VM or run less workers, for instance 1 or 2 instead of 3. I suggest to run 3 masters since this is the minimal number of masters to deploy a high available Kubernetes cluster since etcd needs quorum.

At this point, we can start configuring the baremetal server. Note that these steps are required to be executed by an administrator of the baremetal server (smc-master). First, install all the virtualization packages needed to create virtual machines (VMs), virtual networks, virtual disks and all virtual devices needed to provision the cluster.

```
yum groupinstall "Virtualization Host"
```

Also I suggest tp install the libvirtd client in the baremetal server itself in case we need to troubleshoot locally any issue that can arise.

```
yum install libvirt-client
```

Finally, start and enable the systemd service:

```
systemctl enable libvirtd --now
```

## Libvirt CLI

In order to deploy all the virtual devices needed to run the infrastructure we can make use of the virsh command line. The virsh program is the main interface for managing virsh guest domains. The program can be used to create, pause, and shutdown domains. It can also be used to list current domains.

However, I find much easier to use [kcli])(https://kcli.readthedocs.io/en/master/) to deploy my virtual infrastructure. *Kcli* is a tool meant to interact with existing virtualization providers (libvirt, kubevirt, ovirt, openstack, gcp and aws, vsphere) and to easily deploy and customize vms from cloud images. You can also interact with those vms (list, info, ssh, start, stop, delete, console, serialconsole, add/delete disk, add/delete nic,…). Futhermore, you can deploy vms using predefined profiles, several at once using plan files or entire products for which plans were already created for you.


### Install kcli

Follow the kcli [documentation](https://kcli.readthedocs.io/en/master/#installation) to install and configure all the binaries needed to manage the libvirt daemon of the baremetal server

If using CentOS, which is our case

```
yum install epel-release
yum install pip3 python3 libvirt-python libvirt-devel gcc
pip3 install kcli
```

> It is possible to install kcli on your laptop and configure it to manage the remote baremetal server or even several libvirt hosts remotely. Take a look at the [official documentation](https://kcli.readthedocs.io/en/latest/#configuration). In this tutorial we assume installing kcli on the baremetal server in order to manage the local libvirt.


### Configure kcli to manage the local libvirt

Once you have kcli configured in your baremetal server you need to create a ssh key pair with an empty passphrase to interact with the libvirt daemon. Also this public ssh key will be automatically injected into the virtual machines you are about to create, allowing you to ssh from the baremetal server automatically once the vm is up and running.

```
# ssh-keygen -t rsa -b 2048
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

Establish the connection between the kcli and the local libvirt.

```
kcli create host kvm -H 127.0.0.1
```

Verify

```
+-----------+------+---------+---------+
| Client    | Type | Enabled | Current |
+-----------+------+---------+---------+
| localhost | kvm  |   True  |    X    |
+-----------+------+---------+---------+
```

Next step is create a pool where the cloud images are donwloaded and where the vms are going to be placed in the baremetal server. In this case, we use the defacto libvirt image path:

```
kcli create pool -p /var/lib/libvirt/images/ default
Adding pool default...
```

Finally, gather all the information from the host you already installed:

```
# kcli info host
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

> Note that there is a default network already configured when installing libvirt called default (192.168.122.0/24)

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` followed by `shift+:`. Next type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

> There are other options, the most remarkable in my opinion is [terminator](https://terminator-gtk3.readthedocs.io/en/latest/) which I use a lot.

Next: [Installing the Client Tools](02-client-tools.md)
