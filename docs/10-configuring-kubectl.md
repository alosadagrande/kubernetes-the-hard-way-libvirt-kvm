# Configuring kubectl for Remote Access 

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials. Also, in this lab we are going to show how to provide remote access to the Kubernetes cluster:

- First, configure access to the Kubernetes API from the baremetal server. Remember that it is able to reach the virtual network and any virtual resource since it hosts them.
- Second, in case there is no way to provide ssh access to the baremetal server to your users, it is shown how to configure the baremetal server to forward external Kuberentes API requests to the internal load balancer VM. Note that there is not direct connectivity between external resources to the baremetal server 

> Run the commands in this lab from the same directory used to generate the admin client certificates.

# Remote Access from the baremetal server (host)

Connect to the baremetal server.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```
{
  KUBERNETES_PUBLIC_ADDRESS=$(kcli ssh loadbalancer "hostname --ip-address")

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

## Verification

Check the health of the remote Kubernetes cluster. Remember that all commands are executed from the baremetal server.

Check the status of the components: https://github.com/kubernetes/enhancements/issues/553

```
kubectl get componentstatuses -o yaml | egrep "name:|kind:|message:"

  - message: ok
  kind: ComponentStatus
    name: scheduler
  - message: ok
  kind: ComponentStatus
    name: controller-manager
  - message: '{"health":"true"}'
  kind: ComponentStatus
    name: etcd-0
  - message: '{"health":"true"}'
  kind: ComponentStatus
    name: etcd-1
  - message: '{"health":"true"}'
  kind: ComponentStatus
    name: etcd-2
kind: List
```

List the nodes in the remote Kubernetes cluster:

```
kubectl get nodes
```

> output

```
NAME                     STATUS   ROLES    AGE     VERSION
worker00.k8s-thw.local   Ready    <none>   3h24m   v1.16.2
worker01.k8s-thw.local   Ready    <none>   3h24m   v1.16.2
worker02.k8s-thw.local   Ready    <none>   3h24m   v1.16.2

```

# Remote Acces from outside the baremetal server

In this case we want to give access to someone who is working with his laptop or server and wants to get access to our Kubernetes cluster. Note that this server or laptop must be able to reach the baremetal server where all the VMs are running.


## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Grab the IP address of the baremetal host. Be sure that is resolvable and reachable from the remote user's device.

```
# getent hosts smc-master
10.19.138.41    smc-master.cloud.lab
```

Generate a kubeconfig file suitable for authenticating as the `admin` user:
```

{
REMOTESERVER=smc-master
KUBERNETES_PUBLIC_ADDRESS=$(getent hosts smc-master.cloud.lab | awk '{print$1}' | tr -d '[:space:]')

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${REMOTESERVER}.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=${REMOTESERVER}.kubeconfig

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=${REMOTESERVER}.kubeconfig

  kubectl config use-context kubernetes-the-hard-way --kubeconfig=${REMOTESERVER}.kubeconfig
}
```
> NOTE that from a remote user the entry point of the Kubernetes cluster (KUBERNETES_PUBLIC_ADDRESS) is the baremetal host.

Once you have the kubeconfig file created, we need to expose in some way the loadbalancer VMs to the outside world. Easiest way is to leverage the iptables foo. Basically forwards all requests to the baremetal server port tcp/6443 to the loadbalancer port tcp/6443, which in the same way will redirect the request to any of the three masters.

Note that:

* IP of the loadbalancer VM is  192.168.111.68 
* IP of the baremetal server is 10.19.138.41 
* Libvirt interface of the virtual subnet is k8s-net. Basically, this the interface where the 192.168.111.0/24 subnet is created
* em1 is the interface where the baremetal server is reachable and it is assigned to 10.19.138.41
* 6443/tcp is Kubernetes API server port

Apply the following firewall rules to the baremetal host:

```
    # connections from outside
    $ iptables -I FORWARD -o k8s-net -d  192.168.111.68 -j ACCEPT
    $ iptables -t nat -I PREROUTING -p tcp --dport 6443 -j DNAT --to 192.168.111.68:6443

    # Masquerade local subnet
    $ iptables -I FORWARD -o k8s-net -d  192.168.111.68 -j ACCEPT
    $ iptables -t nat -A POSTROUTING -s 192.168.111.0/24 -j MASQUERADE
    $ iptables -A FORWARD -o k8s-net -m state --state RELATED,ESTABLISHED -j ACCEPT
    $ iptables -A FORWARD -i k8s-net -o em1 -j ACCEPT
    $ iptables -A FORWARD -i k8s-net -o lo -j ACCEPT
```


Then, once applied, you can send the kubeconfig file (${REMOTESERVER}.kubeconfig) to any remote user you want to reach your cluster. 

> NOTE that remote device at least must be able to reach port tcp/6443 of the baremetal server. 

Finally verify from your laptop that you can manage the cluster (see that I am running this command from my laptop @ Katatonic)

```
alosadag@Katatonic $ kubectl get nodes --kubeconfig smc-master.kubeconfig 
NAME                     STATUS   ROLES    AGE     VERSION
worker00.k8s-thw.local   Ready    <none>   4h14m   v1.16.2
worker01.k8s-thw.local   Ready    <none>   4h14m   v1.16.2
worker02.k8s-thw.local   Ready    <none>   4h14m   v1.16.2

```


Next: [Provisioning Pod Network Routes](11-pod-network-routes.md)
