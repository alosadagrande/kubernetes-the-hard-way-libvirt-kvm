# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network routes on virtual servers.

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in all the worker nodes.

Print the internal IP address and Pod CIDR range for each worker instance:

```
for node in worker00 worker01 worker02; do 
	kcli ssh ${node} "hostname -f; hostname -i; cat /home/centos/pod_cidr.txt" | tr '\r\n' ' '
done
```

> output

```
worker00.k8s-thw.local 192.168.111.198 10.200.0.0/24
worker01.k8s-thw.local 192.168.111.253 10.200.1.0/24
worker02.k8s-thw.local 192.168.111.158 10.200.2.0/24
```

With this information you can create the appropiate routes

## Routes

Create network routes for each worker instance:

- worker00

```
[root@worker00 ~]# ip route add 10.200.2.0/24 via 192.168.111.158
[root@worker00 ~]# ip route add 10.200.1.0/24 via 192.168.111.253
```

- worker01

```
[root@worker01 ~]# ip route add 10.200.2.0/24 via 192.168.111.158
[root@worker01 ~]# ip route add 10.200.0.0/24 via 192.168.111.198

```
- worker02

```
[root@worker02 ~]# ip route add 10.200.1.0/24 via 192.168.111.253
[root@worker02 ~]# ip route add 10.200.0.0/24 via 192.168.111.198

```

Verify that the routes were successfully added to the routing table of each worker node. Below is shown the route table of **worker01**:

> output

```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.111.1   0.0.0.0         UG    0      0        0 eth0
10.200.0.0      192.168.111.198 255.255.255.0   UG    0      0        0 eth0
10.200.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cnio0
10.200.2.0      192.168.111.158 255.255.255.0   UG    0      0        0 eth0
169.254.0.0     0.0.0.0         255.255.0.0     U     1002   0        0 eth0
192.168.111.0   0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
