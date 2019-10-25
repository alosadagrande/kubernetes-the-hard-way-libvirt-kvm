# Configuring kubectl for Remote Access 

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

> Run the commands in this lab from the same directory used to generate the admin client certificates.

# Remote Acces from the baremetal server

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```
{
  KUBERNETES_PUBLIC_ADDRESS=$(kcli info vm loadbalancer | grep "ip:" | cut -d ":" -f2  | tr -d '[:space:]')

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

Check the health of the remote Kubernetes cluster:

```
# hostname
smc-master.cloud.lab.eng.bos.redhat.com
# whoami
root
```
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

In this case we want to give access to our cluster to someone who is working with his laptop or server and wants to connect to our cluster. Note that this server or laptop must be able to reach the baremetal server where all the VMs are running.


## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```
# getent hosts smc-master
10.19.138.41    smc-master.cloud.lab.eng.bos.redhat.com
```

```
{
REMOTESERVER=smc-master
KUBERNETES_PUBLIC_ADDRESS=$(getent hosts smc-master | awk '{print$1}')

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

Once you have the kubeconfig file created, we need to expose in some way the loadbalancer VMs to the outside world. Easiest way is to leverage the iptables foo. Basically forwards all requests to the baremetal server port tcp/6443 to the loadbalancer port tcp/6443, which in the same way will redirect the request to any of the three masters.

Note that:

* IP of the loadbalancer VM is  192.168.111.68 
* IP of the baremetal server is 10.19.138.41 
* Libvirt interface of the virtual subnet is k8s-net. Basically the interface where the 192.168.111.0/24 subnet is created
* em1 is the interface where the baremetal server is reachable and it is assigned to 10.19.138.41
* 6443/tcp is Kubernetes API server port

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


Then, you can send this smc-kubeconfig file to any remote user you want to reach your cluster. Note that that at least must be able to reach port tcp/6443 from your baremetal server. Finally verify from your laptop that you can manage the cluster:

```
alosadag@Katatonic $ kubectl get nodes --kubeconfig smc-master.kubeconfig 
NAME                     STATUS   ROLES    AGE     VERSION
worker00.k8s-thw.local   Ready    <none>   4h14m   v1.16.2
worker01.k8s-thw.local   Ready    <none>   4h14m   v1.16.2
worker02.k8s-thw.local   Ready    <none>   4h14m   v1.16.2

```


Next: [Provisioning Pod Network Routes](11-pod-network-routes.md)
