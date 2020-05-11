# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete all the instance and the load balancer:

```
kcli delete plan kubernetes-the-hard-way
```

## Networking

Delete the k8s-net virtual network created

```
kcli delete network k8s-net
```

## Image

Optionally delete the image:

```
kcli delete image --yes CentOS-7-x86_64-GenericCloud.qcow2
```

## Project

Optionally delete the virtualization packages:

```
yum group remove "Virtualization Host"
```
