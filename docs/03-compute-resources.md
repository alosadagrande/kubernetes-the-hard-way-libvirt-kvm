# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Public Network
The public network is a network that contains external access and can be
reached by the outside world. The public network creation can be only done by
an OpenStack administrator.

The following commands provide an example of creating an OpenStack provider
network for public network access.

As an OpenStack administrator:

```
[root@smc-master k8s-th]# kcli list network
Listing Networks...
+---------+--------+------------------+------+---------+------+
| Network |  Type  |       Cidr       | Dhcp |  Domain | Mode |
+---------+--------+------------------+------+---------+------+
| default | routed | 192.168.122.0/24 | True | default | nat  |
+---------+--------+------------------+------+---------+------+

[root@smc-master k8s-th]# kcli create network -c 192.168.111.0/24 k8s-net --domain k8s-thw.local
Network k8s-net deployed
[root@smc-master k8s-th]# kcli list network
Listing Networks...
+---------+--------+------------------+------+---------------+------+
| Network |  Type  |       Cidr       | Dhcp |     Domain    | Mode |
+---------+--------+------------------+------+---------------+------+
| default | routed | 192.168.122.0/24 | True |    default    | nat  |
| k8s-net | routed | 192.168.111.0/24 | True | k8s-thw.local | nat  |
+---------+--------+------------------+------+---------------+------+
```

### Images
To be able to create instances, an image should be provided. Depending on the
OpenStack environment an image catalog can be provided. In this guide we will
use CentOS 7 as the base operating system. Follow the next steps to upload a new
CentOS 7 image:

Download the latest CentOS 7 Generic Cloud image. 

```
[root@smc-master k8s-th]# kcli download image centos7 --pool default
Grabbing image CentOS-7-x86_64-GenericCloud.qcow2...
```

> Basically this is donwloading the latest CentOS 7 cloud image and place it in the default pool we already defined  (/var/lib/libvirt/images/)


### DNS

It is required to have a proper DNS configuration. If not using DNSaaS, you can
create an instance and setup a proper DNS following the next instructions.

Create a security group to allow DNS communication within the internal network:

```
[root@smc-master k8s-th]# getent hosts 192.168.111.68
192.168.111.68  loadbalancer loadbalancer.k8s-net

[root@smc-master k8s-th]# kcli ssh master01 ping loadbalancer
Warning: Permanently added '192.168.111.173' (ECDSA) to the list of known hosts.
PING loadbalancer.k8s-thw.local (192.168.111.68) 56(84) bytes of data.
64 bytes from loadbalancer.k8s-thw.local (192.168.111.68): icmp_seq=1 ttl=64 time=0.370 ms
64 bytes from loadbalancer.k8s-thw.local (192.168.111.68): icmp_seq=2 ttl=64 time=0.280 ms

```

# Configuring SSH Access

SSH will be used to configure the controller and worker instances. The `openstack create` command contained a specific ssh key that has been generated
previously and has been injected in the instances in order to be able to
connect to them without any password prompt.

To avoid asking to add the key to the `~/.ssh/known_hosts` file, the following
snippet will do this up front:

```
for host in $(openstack server list -f value -c Name); do
  ssh-keyscan -H ${host} >> ~/.ssh/known_hosts
done
```

### Load Balancer
In order to have a proper Kubernetes high available environment, a Load balancer
is required to distribute the API load. If not using LBaaS, you can create an
instance and setup a proper Load balancer following the next instructions.

Create an instance to host the load balancer service. In this case the CentOS image is used.

```
# kcli create vm -i centos7 -P disks=[20] -P nets=[k8s-net] -P memory=2048 -P numcpus=2 -P cmds=["yum -y update"] -P reserverdns=yes -P reserverip=yes loadbalancer
```


#### Load balancer service

The following steps shows how to install a load balancer service using HAProxy in the instance previously created.

First, connect to the instance:

```
$ kcli ssh loadbalancer
```

Install and configure HAProxy:

```
# export DOMAIN="k8s-thw.local"
# yum install -y haproxy

# tee /etc/haproxy/haproxy.cfg << EOF
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
    server master00.${DOMAIN} 192.168.111.72:6443 check
    server master01.${DOMAIN} 192.168.111.173:6443 check
    server master02.${DOMAIN} 192.168.111.230:6443 check
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

Finally, update all packages to latest version and reboot the instance just in
case:

```
sudo yum clean all
sudo yum update -y
sudo reboot
```

## Compute Instances

Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
[root@smc-master k8s-th]# for node in master00 master01 master02; do kcli create vm -i centos7 -P disks=[50] -P nets=[k8s-net] -P memory=16384 -P numcpus=4 -P cmds=["yum -y update"] -P reservedns=yes -P reserveip=yes -P reservehost=yes ${node}; done


[root@smc-master k8s-th]# kcli list vm
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
|     Name     | Status |       Ips       |               Source               |  Plan | Profile | Report |
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
| loadbalancer |   up   |  192.168.111.68 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master00   |   up   |  192.168.111.72 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master01   |   up   | 192.168.111.173 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
|   master02   |   up   | 192.168.111.230 | CentOS-7-x86_64-GenericCloud.qcow2 | kvirt | centos7 |        |
+--------------+--------+-----------------+------------------------------------+-------+---------+--------+
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:



```
# for node in worker00 worker01 worker02; do kcli create vm -i centos7 -P disks=[50] -P nets=[k8s-net] -P memory=16384 -P numcpus=4 -P cmds=["yum -y update"] -P reservedns=yes -P reserveip=yes -P reservehost=yes ${node}; done

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

As the DNS server is created internally, it is a good idea to configure an
external DNS server to use the public IPs to avoid using IPs to connect to the
instances.

In this case for the sake of simplicity, we are using the local /etc/hosts in
the local workstation:

```
openstack server list -f value -c Networks -c Name | sed -e 's/kubernetes-the-hard-way=.*, //g' | awk ' { t = $1; $1 = $2; $2 = t; print; } ' | sudo tee -a /etc/hosts
```



Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
