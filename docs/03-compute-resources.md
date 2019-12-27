# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Machines Network

The VM network is a network where all the VMs are executed. Actually this is a virtual network configured as type 'nated', e.g. all VMs will be placed in the same network address range, they will have access to the outside world using the baremetal server virtual bridge as a default gateway (NAT). However, take into account that any remote server won't be able to reach any VM since they are behind the baremetal server.

> Since the baremetal server hosts the virtual infrastructure it is able to connect to any of the VMs. So, as we will see further in this tutorial, anytime you need to execute commands on any of the VMs, you need to connect first to the baremetal server.

By default as commented in the previous sections, there is a default virtual network named as default configured:
```
kcli list network
```
Output expected

```
----+------------------+------+---------+------+
| Network |  Type  |       Cidr       | Dhcp |  Domain | Mode |
+---------+--------+------------------+------+---------+------+
| default | routed | 192.168.122.0/24 | True | default | nat  |
+---------+--------+------------------+------+---------+------+
```

We are going to create a new virtual network to place all the Kubernetes cluster resources:

* The network address range is 192.168.111.0/24
* Name of the network is k8s-net
* The domain of this network is k8s-thw.local. This is important since all VMs in this virtual network will get this domain name as part of its fully qualified name.

```
kcli create network -c 192.168.111.0/24 k8s-net --domain k8s-thw.local
```

Output expected:

```
Network k8s-net deployed
```

Check the list of virtual networks available:

```
# kcli list network
Listing Networks...
+---------+--------+------------------+------+---------------+------+
| Network |  Type  |       Cidr       | Dhcp |     Domain    | Mode |
+---------+--------+------------------+------+---------------+------+
| default | routed | 192.168.122.0/24 | True |    default    | nat  |
| k8s-net | routed | 192.168.111.0/24 | True | k8s-thw.local | nat  |
+---------+--------+------------------+------+---------------+------+
```

## Images

To be able to create instances, an image should be provided. In this guide we will use CentOS 7 as the base operating system for all the VMs. With kcli this is super easy, just download the cloud CentOS 7 image with kcli command line:

```
kcli download image centos7 --pool default
```

> Basically kcli is donwloading the latest CentOS 7 cloud image and placing it in the default pool we already defined  (/var/lib/libvirt/images/)


## DNS

It is required to have a proper DNS configuration that must resolve direct and reverse queries of all the VMs. Unlike other similar tutorials, kcli makes really easy to configure a proper DNS resolution of each VM. Everytime you create a new instance it is possible to create a DNS record into **libvirt dnsmasq** running on the baremetal host. It also can even create a /etc/hosts record in the host that executes the instance creation. This information can be found in the kcli official documentation, section [ip, dns and host reservations](https://kcli.readthedocs.io/en/latest/#ip-dns-and-host-reservations)

> There is no need to maintain a DNS server since DNS record can be automatically created when launching a new instance


## Configuring SSH Access

SSH will be used to configure the loadbalancer, controller and worker instances. By leveraging kcli there is no need to manually exchange the ssh key among all the instances. Kcli automatically injects (using cloudinit) the public ssh key from the baremetal server to all the instances at creation time. Therefore, once the instance is up and running you can easily running `kcli ssh vm_name`


## Compute Instances

Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes **control plane**. Basically we are creating 3 new instances configured with:

- CentOS image as OS
- 50 GB disk
- Connected to the k8s-net (192.168.111.0/24)
- 16GB of memory and 4 vCPus
- Create a DNS record, in this case ${node}.k8s-thw.local which will included in libvirt's dnsmasq (reservedns=yes)
- Reserve the IP, so it is not available to any other VM (reserveip=yes)
- Create an record into baremetal server's /etc/host so it can be reached from outside the virtual network domain as well. (reserveip=yes)
- Execute "yum update -y" once the server is up and running. This command is injected into the cloudinit, so all instances are up to date since the very beginning.

```
# for node in master00 master01 master02; do
 	kcli create vm -i centos7 -P disks=[50] -P nets=[k8s-net] -P memory=16384 -P numcpus=4 \
        -P cmds=["yum -y update"] -P reservedns=yes -P reserveip=yes -P reservehost=yes ${node}
done
```

Verify your masters are up and running

```
[root@smc-master k8s-th]# kcli list vm
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
|     Name     | Status |       Ips       |               Source               |  Plan | Profile | Report |
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
|   master00   |   up   |  192.168.111.72 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master01   |   up   | 192.168.111.173 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master02   |   up   | 192.168.111.230 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
```

### Load Balancer

In order to have a proper Kubernetes high available environment, a Load balancer is required to distribute the API load. In this case we are going to create an specific instance to run a HAProxy loadbalancer service. First, create an instance to host the load balancer service. Below we are about to create a new instance with:

```
# kcli create vm -i centos7 -P disks=[20] -P nets=[k8s-net] -P memory=2048 -P numcpus=2 \
  -P cmds=["yum -y update"] -P reserverdns=yes -P reserverip=yes -P reserverhost=yes loadbalancer
```

Check your **loadbalancer** instance is up and running

```
kcli list vm
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
|     Name     | Status |       Ips       |               Source               |  Plan | Profile | Report |
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
| loadbalancer |   up   |  192.168.111.68 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master00   |   up   |  192.168.111.72 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master01   |   up   | 192.168.111.173 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master02   |   up   | 192.168.111.230 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
```
#### Load balancer service

The following steps shows how to install a load balancer service using HAProxy in the instance previously created.

First, connect to the instance:

```
kcli ssh loadbalancer
```

Install and configure HAProxy:

```
# export DOMAIN="k8s-thw.local"
# yum install -y haproxy bind-utils
# for node in master00 master01 master02 ; do export IP_${node}="$(dig +short ${node})"; done
 

# sudo tee /etc/haproxy/haproxy.cfg << EOF
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats

defaults
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen stats :9000
    stats enable
    stats realm Haproxy\ Statistics
    stats uri /haproxy_stats
    stats auth admin:password
    stats refresh 30
    mode http

frontend  main *:6443
    default_backend mgmt6443
    option tcplog

backend mgmt6443
    balance source
    mode tcp
    # MASTERS 6443
    server master00.${DOMAIN} ${IP_master00}:6443 check
    server master01.${DOMAIN} ${IP_master01}:6443 check
    server master02.${DOMAIN} ${IP_master02}:6443 check
EOF
```

As the Kubernetes port is 6443, the selinux policy should be modified to allow
haproxy to listen on that particular port:

```
sudo semanage port --add --type http_port_t --proto tcp 6443
```

Verify everything is properly configured:

```
haproxy -c -V -f /etc/haproxy/haproxy.cfg
```

Start and enable the service

```
sudo systemctl enable haproxy --now
```


### Kubernetes Workers

Create three compute instances which will host the Kubernetes worker nodes:

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

> Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `/home/centos/pod_cidr.txt` file contains the subnet assigned to each worker.

```
# kcli create vm -i centos7 -P disks=[50] -P nets=[k8s-net] -P memory=16384 -P numcpus=4 \
 -P cmds=["yum -y update",'echo "10.200.0.0/24" > /home/centos/pod_cidr.txt'] -P reservedns=yes -P reserveip=yes -P reservehost=yes worker00

# kcli create vm -i centos7 -P disks=[50] -P nets=[k8s-net] -P memory=16384 -P numcpus=4 \
  -P cmds=["yum -y update",'echo "10.200.1.0/24" > /home/centos/pod_cidr.txt'] -P reservedns=yes -P reserveip=yes -P reservehost=yes worker01
# kcli create vm -i centos7 -P disks=[50] -P nets=[k8s-net] -P memory=16384 -P numcpus=4 \
-P cmds=["yum -y update",'echo "10.200.2.0/24" > /home/centos/pod_cidr.txt'] -P reservedns=yes -P reserveip=yes -P reservehost=yes worker02
```

### Verification

List the compute instances:

```
# kcli list vm
```

> output

```
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
|     Name     | Status |       Ips       |               Source               |  Plan | Profile | Report |
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
| loadbalancer |   up   |  192.168.111.68 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master00   |   up   |  192.168.111.72 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master01   |   up   | 192.168.111.173 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master02   |   up   | 192.168.111.230 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   worker00   |   up   | 192.168.111.198 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   worker01   |   up   | 192.168.111.253 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   worker02   |   up   | 192.168.111.158 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
```

## DNS Verification

Once all the instances are deployed, we need to verify that the DNS records are correctly configured before starting the Kubernetes cluster installation. Verify instances are resolved in the baremetal server, note that the records were stored in the /etc/hosts

```
getent hosts 192.168.111.68
```

Output

```
192.168.111.68  loadbalancer loadbalancer.k8s-net
```
Content of the baremetal server /etc/hosts should be similar to the following:

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.111.68 loadbalancer loadbalancer.k8s-thw.local # KVIRT
192.168.111.72 master00 master00.k8s-thw.local # KVIRT
192.168.111.173 master01 master01.k8s-thw.local # KVIRT
192.168.111.230 master02 master02.k8s-thw.local # KVIRT
192.168.111.198 worker00 worker00.k8s-thw.local # KVIRT
192.168.111.253 worker01 worker01.k8s-thw.local # KVIRT
192.168.111.158 worker02 worker02.k8s-thw.local # KVIRT
```

Finally, verify that each instance is able to resolve another instance hostname. As shown below, master01 is able to resolved loadbalancer hostname:

```
kcli ssh master01 ping loadbalancer
```

Output expected:

```
Warning: Permanently added '192.168.111.173' (ECDSA) to the list of known hosts.
PING loadbalancer.k8s-thw.local (192.168.111.68) 56(84) bytes of data.
64 bytes from loadbalancer.k8s-thw.local (192.168.111.68): icmp_seq=1 ttl=64 time=0.370 ms
64 bytes from loadbalancer.k8s-thw.local (192.168.111.68): icmp_seq=2 ttl=64 time=0.280 ms
```

## Reboot

Finally, since all packages were updated during the bootstrap of the instance. we must reboot to run the latest ones

```
for node in loadbalancer master00 master01 master02 worker00 worker01 worker02
do
	kcli restart vm $node
done
```


Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
