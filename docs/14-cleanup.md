# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
openstack server delete \
  controller-0.${DOMAIN} controller-1.${DOMAIN} controller-2.${DOMAIN} \
  worker-0.${DOMAIN} worker-1.${DOMAIN} worker-2.${DOMAIN}
```

Optionally, delete the DNS and Load Balancer instances:

```
openstack server delete \
  k8sosp.${DOMAIN} dns.${DOMAIN}
```

## Networking

Delete the `kubernetes-the-hard-way` security groups:

```
openstack security group delete \
  kubernetes-the-hard-way-allow-internal \
  kubernetes-the-hard-way-allow-external \
  kubernetes-the-hard-way-allow-dns
```

Delete floating IPs:

```
openstack floating ip delete $(openstack floating ip list -f value -c ID)
```

Detach the router and subnet:

```
openstack router remove subnet kubernetes-the-hard-way-router kubernetes
openstack router unset --external-gateway kubernetes-the-hard-way-router
```

Delete unused ports just in case:

```
for PORT in $(openstack port list --router kubernetes-the-hard-way-router --format=value -c ID)
do
  openstack router remove port kubernetes-the-hard-way-router $PORT
done
```

Delete the router, subnet and network:

```
{
  openstack router delete kubernetes-the-hard-way-router

  openstack subnet delete kubernetes

  openstack network delete kubernetes-the-hard-way
}
```

## Image

Optionally delete the image:

```
openstack image delete CentOS-7-x86_64-GenericCloud-1907
```

## Project

Optionally delete the project:

```
openstack project delete kubernetes-the-hard-way
```
