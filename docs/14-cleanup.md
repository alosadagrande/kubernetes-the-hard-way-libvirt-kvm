# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
kcli delete vm \
  master00 master01 master02 \
  worker00 worker01 worker02
```

Optionally, delete the load balancer instance:

```
kcli delete vm loadbalancer
```

## Networking

Delete the k8s-net virtual network created

```
kcli delete network k8s-net
```

## Image

Optionally delete the image:

```
rm -rf /var/lib/libvirt/images/CentOS-7-x86_64-GenericCloud.qcow2
```

## Project

Optionally delete the virtualization packages:

```
yum group remove "Virtualization Host"
```
